---
title: "Java安全策略配置和沙箱闲话"
date: 2021-06-17T08:39:26+08:00
tags: [Java]
---

之前一篇文章提到了System.exit和SecurityManager,引入了下面的代码
```java
public class SelfSecurityManager extends SecurityManager{
	..//
   @Override
    public void checkExit(int status) {
        super.checkExit(status);
        throw new ExitException(status);
    }
}
```
通过自定义SecurityManager来禁止System.exit的执行，这里我们来分析下其实现原理，看下super.checkExit方法
```java
	/*
     * @param      status   the exit status.
     * @exception SecurityException if the calling thread does not have
     *              permission to halt the Java Virtual Machine with
     *              the specified status.
     * @see        java.lang.Runtime#exit(int) exit
     * @see        #checkPermission(java.security.Permission) checkPermission
     */
    public void checkExit(int status) {
        checkPermission(new RuntimePermission("exitVM."+status));
    }
```
可以看到checkPermission方法检查了RuntimePermission类型中的exitVM操作。RuntimePermission是JVM对运行时权限的一个集合，提供了对Java运行时的权限定义，那RuntimePermission有哪些权限呢，我们可以看下RuntimePermission类上的注释，或者直接看官方文档

## Permission

可以看到仅仅是RuntimePermission类型就非常之多 ，包括之前我们提到的setSecurityManager和exitVM。

| Permission Target Name           | What the Permission Allows                                   | Risks of Allowing this Permission                            |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| createClassLoader                | Creation of a class loader                                   | This is an extremely dangerous permission to grant. Malicious applications that can instantiate their own class loaders could then load their own rogue classes into the system. These newly loaded classes could be placed into any protection domain by the class loader, thereby automatically granting the classes the permissions for that domain. |
| getClassLoader                   | Retrieval of a class loader (e.g., the class loader for the calling class) | This would grant an attacker permission to get the class loader for a particular class. This is dangerous because having access to a class's class loader allows the attacker to load other classes available to that class loader. The attacker would typically otherwise not have access to those classes. |
| setContextClassLoader            | Setting of the context class loader used by a thread         | The context class loader is used by system code and extensions when they need to lookup resources that might not exist in the system class loader. Granting setContextClassLoader permission would allow code to change which context class loader is used for a particular thread, including system threads. |
| enableContextClassLoaderOverride | Subclass implementation of the thread context class loader methods | The context class loader is used by system code and extensions when they need to lookup resources that might not exist in the system class loader. Granting enableContextClassLoaderOverride permission would allow a subclass of Thread to override the methods that are used to get or set the context class loader for a particular thread. |
| closeClassLoader                 | Closing of a ClassLoader                                     | Granting this permission allows code to close any URLClassLoader that it has a reference to. |
| setSecurityManager               | Setting of the security manager (possibly replacing an existing one) | The security manager is a class that allows applications to implement a security policy. Granting the setSecurityManager permission would allow code to change which security manager is used by installing a different, possibly less restrictive security manager, thereby bypassing checks that would have been enforced by the original security manager. |
| createSecurityManager            | Creation of a new security manager                           | This gives code access to protected, sensitive methods that may disclose information about other classes or the execution stack. |
| getenv.{variable name}           | Reading of the value of the specified environment variable   | This would allow code to read the value, or determine the       existence, of a particular environment variable.  This is       dangerous if the variable contains confidential data. |
| exitVM.{exit status}             | Halting of the Java Virtual Machine with the specified exit status | This allows an attacker to mount a denial-of-service attack by automatically forcing the virtual machine to halt. Note: The "exitVM.*" permission is automatically granted to all code loaded from the application class path, thus enabling applications to terminate themselves. Also, the "exitVM" permission is equivalent to "exitVM.*". |
| shutdownHooks                    | Registration and cancellation of virtual-machine shutdown hooks | This allows an attacker to register a malicious shutdown hook that interferes with the clean shutdown of the virtual machine. |
|..|..|..|

也就是说，Java代码里，禁止执行哪些方法，允许执行哪些方法，都是有开关可以控制的。除了RuntimePermission，还有 AudioPermission, AuthPermission, AWTPermission, DelegationPermission, JAXBPermission, LinkPermission, LoggingPermission, ManagementPermission, MBeanServerPermission, MBeanTrustPermission, NetPermission, PropertyPermission, ReflectPermission, RuntimePermission, SecurityPermission, SerializablePermission, SQLPermission, SSLPermission, SubjectDelegationPermission, WebServicePermission等类型，比如设置Java代码对操作系统和JVM常量读取的PropertyPermission，设置读写文件位置和操作的FilePermission等。

需要说明的是，RuntimePermission是java.security.BasicPermission的子类，BasicPermission又是java.security.Permission的子类，因此FilePermission和RuntimePermission不在一个同级别的包下。

要让这些Permission生效，我们需要在JVM启动时就指定对应的策略文件，比如
java -jar aa.jar -Djava.security.manager -Djava.security.policy==d:/other/security.policy

## policy策略文件

我们接着讲下policy策略文件是啥。

policy文件的作用是指定哪些类有哪些权限。policy文件根据类的url和类的签名来确定类，指定权限，例如：
```bash
grant codeBase "file:${{java.ext.dirs}}/*" {
        permission java.security.AllPermission;
};
grant {
    permission java.io.FilePermission "<<ALL FILES>>", "execute";
};
```
policy文件的具体语法可以看oracle的文档，DSL如下：
```bash
grant signedBy "signer_names", codeBase "URL",
        principal principal_class_name "principal_name",
        principal principal_class_name "principal_name",
        ... {

      permission permission_class_name "target_name", "action", 
          signedBy "signer_names";
      permission permission_class_name "target_name", "action", 
          signedBy "signer_names";
      ...
  };
```

target有限支持 "*" 和"-"这种通配符，但使用场景有限。

一旦开启了policy策略，JVM的权限检查就变成了白名单模式。默认情况下，JVM的policy是AllPermission的（默认的policy文件位于java.home/lib/security/java.policy），如果你定义了自己的policy文件覆盖JVM的默认策略，那么就严格按照你的定义来走了，不在你policy策略里的代码一律无法执行。

```java
import java.io.FileWriter;
import java.io.IOException;

public class Poc {
    public static void main(String[] args) throws IOException {
        exec("calc");
        System.setSecurityManager(null);
        write();
    }

    public static void write() {
        final String str = "Hi ";
        try {
            FileWriter writer = new FileWriter("d:\\a.txt");
            writer.write(str);
            writer.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void exec(String command) {
        try {
            Runtime.getRuntime().exec(command);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
脚本小子要想再执行exec弹个计算器就不行了，异常如下
```java
Launcher failed - "Dump Threads" and "Exit" actions are unavailable (access denied ("java.lang.RuntimePermission" "loadLibrary.G:\idea\bin\breakgen64.dll"))
java.security.AccessControlException: access denied ("java.io.FilePermission" "<<ALL FILES>>" "execute")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
```
可以看到execute被阻止了，甚至连idea的javaagent挂载都被禁止了。因为前面说过，policy策略是白名单模式。



## 实用性

想要把policy策略用起来，看来是很困难的，因为默认的白名单模式也会导致很多常规操作受阻，包括文件读写，一些反射API。要想白名单起作用，需要严格的测试和复杂的policy策略，当然为了安全，这些都是值得的。

在java中，反射是一个常见的操作，如果由于业务需要，无法禁用反射，尤其是Spring，对反射强依赖，这种情况可以设置禁止反射的方法和变量的黑名单，方法就是sun.reflect.Reflection类的registerFieldsToFilter和registerMethodsToFilter方法。

那么policy策略有黑名单模式吗？没有。如果你非要有，也可以自定义java.security.manager类，但是黑名单策略明显是比白名单安全隐患要大的。

要实现安全和易用之间的平衡，还是很难的。另外，Java体系的API是比较复杂的，反射，自定义ClassLoader，loadLibrary等非常规操作都是可以被利用来绕过java security manager的，尤其需要注意。当然，还得从代码和服务器层面来做好安全防控。

java policy策略这么复杂，有什么工具能帮助我们配置吗？这个真的有，那就是JDK自带的policytool，界面如图。
![policytool](https://i.loli.net/2021/06/16/Y26plHAkZfz18R7.png)


如果要在springboot里配置一个policy，会比较复杂，可能会多大几百条指令，这里仅给出一个最小的支持数据库连接的springboot的policy文件

```properties
grant codeBase "file:G:/data/project/bcdemo/target/bcdemo-0.0.1-SNAPSHOT.jar" {
		permission java.util.PropertyPermission "*", "read";
		//
		permission java.util.PropertyPermission "java.awt.headless", "read,write";
		permission java.util.PropertyPermission "spring.beaninfo.ignore", "read,write";
		permission java.util.PropertyPermission "org.graalvm.nativeimage.imagecode", "read";
		permission java.util.PropertyPermission "user.dir", "read";
		permission java.util.PropertyPermission "java.protocol.handler.pkgs", "read,write";
		permission java.util.PropertyPermission "PID", "write";
		permission java.util.PropertyPermission "catalina.*", "read,write";
		permission java.lang.reflect.ReflectPermission "suppressAccessChecks", "read";
		permission java.io.FilePermission "${java.io.tmpdir}/-", "read,write,delete";
		permission java.io.FilePermission "./config/application.properties", "read";
		permission java.io.FilePermission "./config/-", "read";
		permission java.io.FilePermission "./application.properties", "read";
		permission java.io.FilePermission "./application.xml", "read";
		permission java.io.FilePermission "./application.yml", "read";
		permission java.io.FilePermission "./application.yaml", "read";
		permission java.io.FilePermission "./-", "read";
		permission java.io.FilePermission "G:/data/project/bcdemo/target/", "read";
		permission java.io.FilePermission "C:/-", "read";
		
		
		permission java.lang.RuntimePermission "getenv.SPRING_BOOT_ENABLEAUTOCONFIGURATION", "read";
		permission java.lang.RuntimePermission "getenv.spring.boot.enableautoconfiguration", "read";
		permission java.lang.RuntimePermission "getenv.spring.application.name", "read";
		permission java.lang.RuntimePermission "getenv.spring.*", "read";
		permission java.lang.RuntimePermission "getenv.logging.*", "read";
		permission java.lang.RuntimePermission "getenv.LOGGING_REGISTER_SHUTDOWN_HOOK", "read";
		permission java.lang.RuntimePermission "getenv.*", "read";
		permission java.lang.RuntimePermission "createClassLoader", "read";
		permission java.lang.RuntimePermission "setContextClassLoader", "read";
		permission java.lang.RuntimePermission "accessDeclaredMembers", "read";
		permission java.net.SocketPermission "localhost:9010", "listen";
		permission java.net.SocketPermission "127.0.0.1:1024-", "accept,resolve";
		//db
		permission java.net.SocketPermission "10.180.249.85:3306", "connect,resolve";
		permission java.lang.RuntimePermission "getProtectionDomain", "read";
		permission java.lang.RuntimePermission "setFactory", "read";
		permission java.lang.RuntimePermission "accessClassInPackage.sun.reflect", "read";
		permission java.lang.RuntimePermission "getClassLoader", "read";
		permission java.net.NetPermission "specifyStreamHandler", "read";
		
		permission javax.security.auth.AuthPermission "doAsPrivileged", "read";
		permission java.security.SecurityPermission "getProperty.authconfigprovider.factory", "read";
		permission java.lang.RuntimePermission "modifyThread", "read";
		
};
```
这份配置仅供参考，如果你有引入更多依赖或外部连接，那就需要根据报错添加对应策略。



## 一点闲话

java policy有啥实际应用吗？百度应用引擎BAE (Baidu App Engine) 大概是2011年或更早搞的，是模仿SAE（新浪云,2009年11月推出）的一个产品。至于GAE，就不提了，比它们不知高了几个level。

不过SAE那时逼格比较高，刚开始只支持PHP，而且得邀请制开通，还搞了什么新浪豆机制，所以我用的是免费BAE。继续闲话，当时新浪搞了个SAE认证，会玩SAE的颁发个初中高级SAE开发者认证，据说面试可以加分，然后产生了SAE认证一条龙产业，花钱买认证，甚至有人给自己的不识字的父母也搞了个SAE中级认证。。。别笑，是真的。这个圈子可把老哥我当年给整乐了，而且一圈人一直在整乐子，整到了2021年。

PHP比较简单就一句略过，就是Nginx虚拟主机+自定义php.ini那套，说下Java沙箱。BAE对Java的支持也比较有意思，BAE那个年代有kvm和虚拟机了（但是还没有docker），但BAE也好，SAE也好，使用的并不是虚拟技术，同样是更接近虚拟主机的方案。其中BAE对Java的支持我记得是通过SVN提交代码文件到一个固定的目录下就可以了，操作体验非常接近虚拟主机。

那脚本小子就来劲了啊，我得给你穿了，访问别人的文件或系统的文件。还真有人这么做了，穿透BAE/SAE的沙箱读取系统配置文件或者弹个计算器。BAE/SAE当时都是用的java policy搞的Java沙箱技术来实现隔离的，一样有人用反射来绕过沙箱。当然，沙箱实现还不止这种技术，还包括网络的隔离等。

不过现在都2021年了，docker和K8S的广泛使用让Java沙箱的用武之地已经不是那么很大了，docker的隔离也更彻底了，沙箱或许已经成为了历史。