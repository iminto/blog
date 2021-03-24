---
title: "Keycloak接入自研系统"
date: 2021-03-24T17:39:00+08:00
tags: [Java]
---

## 简介
keycloak是一个非常强大的权限认证系统，我们使用keycloak可以方便的实现SSO的功能。虽然keycloak底层使用的wildfly，并且提供了非常方便的Client Adapters和各种服务器进行对接，比如wildfly，tomcat，Jetty等。

对于最流行的SpringBoot来说，keycloak有官方Adapter，只需要修改配置即可。如果非SpringBoot应用呢，那就只能使用Java Servlet Filter Adapter了。

SpringBoot接入keycloak的例子比较多，我就不赘述了。我们有一个老系统，用的embedded jetty+jersey，虽然官方提供了Jetty 9.x Adapters，但这是针对standalone而言的，现在几乎没人这么用了，所以还是得自己来。官方有Java Servlet Filter Adapter的教程，但是用的是web.xml的例子，而且语焉不详，所以这里就我自己的摸索提供一点参考。

## jetty+jersey
先来看一下jetty+jersey的原生例子，涉及两个文件
App.java

```java
package xyz.chen;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
import org.glassfish.jersey.servlet.ServletContainer;

public class App {
    public static void main(String[] args) {
        Server server = new Server(9999);
        ServletContextHandler context = new ServletContextHandler(ServletContextHandler.NO_SESSIONS);
        context.setContextPath("/");
        server.setHandler(context);

        // 配置Servlet
        ServletHolder holder = context.addServlet(ServletContainer.class.getCanonicalName(), "/rest/*");
        holder.setInitOrder(1);
        holder.setInitParameter("jersey.config.server.provider.packages", "xyz.chen");
        holder.setInitParameter("jersey.config.server.provider.classnames", "org.glassfish.jersey.server.filter.CsrfProtectionFilter");

        try {
            server.start();
            server.join();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            server.destroy();
        }
    }
}
```
然后是业务类：
HelloResource.java

```java
package xyz.chen;

import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.MultivaluedMap;
import javax.ws.rs.core.PathSegment;
import javax.ws.rs.core.Response;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

@Path("hello")
public class HelloResource {

    @Path("index")
    @GET
    @Consumes(MediaType.TEXT_PLAIN)
    @Produces(MediaType.TEXT_PLAIN)
    public Response helloworld() {
        return Response.ok("hello jersey").build();
    }

    @GET
    @Path("/user/{userName}")
    public Response getThemeCss(@PathParam("userName") String userName) {

        StringBuilder sb = new StringBuilder(userName);
        return Response.ok(sb.toString()).build();
    }
}
```
pom.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.example</groupId>
  <artifactId>jersey-demo</artifactId>
  <version>1.0-SNAPSHOT</version>

  <name>Maven</name>
  <url>http://maven.apache.org/</url>
  <inceptionYear>2001</inceptionYear>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.glassfish.jersey.core</groupId>
      <artifactId>jersey-server</artifactId>
      <version>2.27</version>
    </dependency>
    <dependency>
      <groupId>org.glassfish.jersey.inject</groupId>
      <artifactId>jersey-hk2</artifactId>
      <version>2.27</version>
    </dependency>
    <dependency>
      <groupId>org.glassfish.jersey.containers</groupId>
      <artifactId>jersey-container-servlet-core</artifactId>
      <version>2.27</version>
    </dependency>
    <dependency>
      <groupId>org.glassfish.jersey.containers</groupId>
      <artifactId>jersey-container-jetty-http</artifactId>
      <version>2.27</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty</groupId>
      <artifactId>jetty-server</artifactId>
      <version>9.4.12.v20180830</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty</groupId>
      <artifactId>jetty-servlet</artifactId>
      <version>9.4.12.v20180830</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty</groupId>
      <artifactId>jetty-util</artifactId>
      <version>9.4.12.v20180830</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>2.4.3</version>
        <configuration>
          <createDependencyReducedPom>true</createDependencyReducedPom>
          <filters>
            <filter>
              <artifact>*:*</artifact>
              <excludes>
                <exclude>META-INF/*.SF</exclude>
                <exclude>META-INF/*.DSA</exclude>
                <exclude>META-INF/*.RSA</exclude>
              </excludes>
            </filter>
          </filters>
        </configuration>

        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <transformers>
                <transformer
                        implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
                <transformer
                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <manifestEntries>
                    <Main-Class>xyz.chen.App</Main-Class>
                  </manifestEntries>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>

```
运行就不再举例了。

## 整合keycloak
引入依赖
```xml
    <dependency>
      <groupId>org.keycloak</groupId>
      <artifactId>keycloak-servlet-filter-adapter</artifactId>
      <version>11.0.2</version>
    </dependency>
```
修改App类，加入Filter
```java
 ServletHandler handler = new ServletHandler();
 FilterHolder fh=handler.addFilterWithMapping(org.keycloak.adapters.servlet.KeycloakOIDCFilter.class,"/*", EnumSet.of(DispatcherType.REQUEST));
fh.setInitParameter("keycloak.config.file", "keycloak.json");
context.addFilter(fh, "/*", EnumSet.of(DispatcherType.REQUEST));
server.setHandler(context);
```
其中，keycloak.json文件来自keycloak-clients-installtion，形如
```json
{
  "realm": "wildfly",
  "auth-server-url": "http://localhost:8080/auth/",
  "ssl-required": "external",
  "resource": "product-app",
  "public-client": true,
  "confidential-port": 0
}
```
keycloak的配置不再赘述。



## 获取用户权限信息

这块不再举例，自己写个Filter去KeycloakSecurityContext里拿就可以了。



## 参考

https://www.keycloak.org/docs/latest/securing_apps/#_jetty9_adapter

https://stackoverflow.com/questions/22188285/does-embedded-jetty-have-the-ability-to-set-the-init-params-of-a-filter