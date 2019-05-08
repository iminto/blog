---
title: 使用内嵌undertow开发调试jfinal项目
date: 2018-05-19 23:56:37
tags: [Java]
---
今天在修一个老项目，使用的是jfinal框架，这个框架算是一个比较传统的框架，只支持打包成war运行放入容器中运行，但是在开发过程中可以使用jetty快速启动和调试。个人不是很喜欢jetty，遂换成了undertow。
引入如下依赖
```xml
<dependency>
	<groupId>io.undertow</groupId>
	<artifactId>undertow-core</artifactId>
	<version>2.0.1.Final</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>io.undertow</groupId>
	<artifactId>undertow-servlet</artifactId>
	<version>2.0.1.Final</version>
	<scope>provided</scope>
</dependency>
```
再写一个启动类就好了
```java
public class Main {
    public static void main(String[] args) throws Exception {
        DeploymentInfo servletBuilder = Servlets.deployment()
                .setContextPath("/")
                .setClassLoader(Main.class.getClassLoader())
                .setDeploymentName("zooadmin.war")
                ;
        FilterInfo jfinalFilter=new FilterInfo("jfinal",JFinalFilter.class);
        jfinalFilter.addInitParam("configClass","com.baicai.core.Config");
        servletBuilder.addFilter(jfinalFilter);
        servletBuilder.addFilterUrlMapping("jfinal","/*", DispatcherType.REQUEST);
        servletBuilder.addFilterUrlMapping("jfinal","/*", DispatcherType.FORWARD);
        servletBuilder.setResourceManager(new FileResourceManager(new File("src/main/webapp"), 1024));

        DeploymentManager manager = Servlets.defaultContainer().addDeployment(servletBuilder);
        manager.deploy();
        PathHandler path = Handlers.path(Handlers.redirect("/"))
               .addPrefixPath("/", manager.start());
        Undertow server = Undertow.builder()
                .addHttpListener(1080, "localhost")
                .setHandler(path)
                .build();
        // start server
        server.start();
    }
}
```
直接在这个类上运行main方法即可。关键的地方就是把传统的web项目的web.xml翻译成Java代码而已。
本来想继续实现springboot那种fatjar的打包方式，最后发现现有的maven插件都无法满足需求，spring是自己扩展了jar的一套协议实现的，实现起来颇有难度。留待以后折腾吧
