---
title: 使用Java自动登录需要动态密码的堡垒机
date: 2018-11-16 20:45:00
tags: [Java,Linux]
---
公司的生产服务器买了QiZhi Technologie的堡垒机，每次登录都得输入密码+空格+OTOP验证码，都得打开手机APP操作一把，烦不胜烦。

不可忍，想了想，还是借助Java在每次调用时自动生成验证码，然后搞个ssh自动登录（别问我问啥不用公钥，哪有权限啊）得了。

结合之前写的博客 [TOTP算法Java版本](http://www.baidecai.com/article/TOTPByJava)，很容易就写出计算验证码的代码：
```java
public long getCode(String secret, long timeMsec) throws Exception {
        Base32 codec = new Base32();
        byte[] decodedKey = codec.decode(secret);
        long t = (timeMsec / 1000L) / 30L;
        for (int i = -window_size; i <= window_size; ++i) {
            long hash;
            try {
                hash = verify_code(decodedKey, t + i);
                return hash;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return 0L;
    }
```
写一个类，专门调用这个方法生成验证码，获取程序执行结果

```java
java -Dfile.encoding=UTF-8 -classpath /soft/tool/authcode/ GoogleAuthTest
```
，接下来，要实现自动登录就简单多了，先写一个shell

```c
#!/bin/bash
passwd=$(java -Dfile.encoding=UTF-8 -classpath /soft/tool/authcode/ GoogleAuthTest)
./prod.exp $passwd
```
shell调用java生成验证码，然后传给expect脚本

```c
#!/bin/expect
set timeout 10
set fullpasswd [lindex $argv 0]
spawn ssh -l chenwen 172.10.3.110
expect "*ssword*"
send "dev744988 $fullpasswd\r"
interact
```
不到100行新代码，搞定收工，全程不到半小时。最耗时的还是传递变量给expact花了不少时间。
