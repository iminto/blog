---
title: "Keycloak接入自研系统"
date: 2021-03-24T17:39:00+08:00
tags: [Java]
---

## 简介
keycloak是一个非常强大的权限认证系统，我们使用keycloak可以方便的实现SSO的功能。虽然keycloak底层使用的wildfly，但是提供了非常方便的Client Adapters和各种服务器进行对接，比如wildfly，tomcat，Jetty等。

对于最流行的SpringBoot来说，keycloak有官方Adapter，只需要修改配置即可。如果非SpringBoot应用呢，那就只能使用Java Servlet Filter Adapter了。

SpringBoot接入keycloak的例子比较多，我就不赘述了。这里只简单说明下。

## 接入前的前置准备

在接入各种应用之前，需要在keycloak中做好相应的配置。一般来说需要使用下面的步骤：

1. 创建新的realm

   一般来说，为了隔离不同类型的系统，我们建议为不同的client创建不同的realm。当然，如果这些client是相关联的，则可以创建在同一个realm中。

2. 创建新的用户和角色。

   用户是用来登录keycloak用的，如果是不同的realm，则需要分别创建用户。用户密码也是在这一步创建的

3. 添加和配置client

   这一步是非常重要的，我们需要根据应用程序的不同，配置不同的root URL，redirect URI等。

   还可以配置mapper和scope信息。

   最后，如果是服务器端的配置的话，还需要installation的一些信息。

   有了这些基本的配置之后，我们就可以准备接入应用程序了。

## Springboot接入keycloak

### 引入依赖

```xml
	<dependency>
		<groupId>org.keycloak</groupId>
		<artifactId>keycloak-spring-boot-starter</artifactId>
		<version>11.0.2</version>
	</dependency>
```
### 然后配置application.yml

```yaml
keycloak:
    auth-server-url: http://localhost:8080/auth
    realm: wildfly
    public-client: true
    resource: product-app
    securityConstraints:
        - authRoles:
              # 以下路径需要user角色才能访问
              - user
          securityCollections:
              # name可以随便写
              - name: user-role-mappings
                patterns:
                    - /users/*


```

至此就接入完成了，不需要编写任何代码。

### 获取KeycloakSecurityContext

但是实际使用中，光能控制登陆权限还不够，业务代码中还需要能获取到当前角色，用户名等信息，这就需要用到KeycloakSecurityContext了。KeycloakSecurityContext是keycloak的上下文，我们可以从其中获取到AccessToken，IDToken，AuthorizationContext和realm信息。

Identity.java

```java
import java.util.List;

import org.keycloak.AuthorizationContext;
import org.keycloak.KeycloakSecurityContext;
import org.keycloak.representations.idm.authorization.Permission;

/**
 * <p>This is a simple facade to obtain information from authenticated users. You should see usages of instances of this class when
 * rendering the home page (@code home.ftl).
 *
 * <p>Instances of this class are are added to models as attributes in order to make them available to templates.
 *
 * @author <a href="mailto:psilva@redhat.com">Pedro Igor</a>
 * @see com.github.your.demo.controller.HomeController
 */
public class Identity {

    private final KeycloakSecurityContext securityContext;

    public Identity(KeycloakSecurityContext securityContext) {
        this.securityContext = securityContext;
    }

    /**
     * An example on how you can use the {@link org.keycloak.AuthorizationContext} to check for permissions granted by Keycloak for a particular user.
     *
     * @param name the name of the resource
     * @return true if user has was granted with a permission for the given resource. Otherwise, false.
     */
    public boolean hasResourcePermission(String name) {
        return getAuthorizationContext().hasResourcePermission(name);
    }

    /**
     * An example on how you can use {@link KeycloakSecurityContext} to obtain information about user's identity.
     *
     * @return the user name
     */
    public String getName() {
        return securityContext.getIdToken().getPreferredUsername();
    }

    /**
     * An example on how you can use the {@link org.keycloak.AuthorizationContext} to obtain all permissions granted for a particular user.
     *
     * @return
     */
    public List<Permission> getPermissions() {
        return getAuthorizationContext().getPermissions();
    }

    /**
     * Returns a {@link AuthorizationContext} instance holding all permissions granted for an user. The instance is build based on
     * the permissions returned by Keycloak. For this particular application, we use the Entitlement API to obtain permissions for every single
     * resource on the server.
     *
     * @return
     */
    private AuthorizationContext getAuthorizationContext() {
        return securityContext.getAuthorizationContext();
    }
}
```

使用

```java
@RestController
public class HomeController {
    private Logger logger = LoggerFactory.getLogger(HomeController.class);
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private HttpServletRequest request;

    @RequestMapping("/users")
    @ResponseBody
    public List<Users> users() {
        logIdentity();
        logger.info("使用JdbcTemplate查询数据库");
        String sql = "SELECT * FROM users ";
        List<Users> queryAllList = jdbcTemplate.query(sql, new Object[]{},
                new BeanPropertyRowMapper<>(Users.class));
        logger.info("查询用户列表" + queryAllList);
        return queryAllList;
    }

    @RequestMapping("/")
    public String home() {
        return "Hello Docker World";
    }

    private void logIdentity() {
        KeycloakSecurityContext context=getKeycloakSecurityContext();
        if(context!=null){
            Identity identity=new Identity(context);
            logger.info("KeycloakSecurityContext identity={}",identity);
        }else{
            logger.info("KeycloakSecurityContext is null");
        }
    }

    private KeycloakSecurityContext getKeycloakSecurityContext() {
        return (KeycloakSecurityContext) request.getAttribute(KeycloakSecurityContext.class.getName());
    }
}
```

Identity类来自Keycloak的官方example。上面介绍的Spring Boot中的其实是隐藏的做法，adaptor自动为我们做了和Keycloak认证服务连接的事情，如果我们需要手动去处理，则需要用到Authorization Client Java API。

添加maven依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.keycloak</groupId>
        <artifactId>keycloak-authz-client</artifactId>
        <version>${KEYCLOAK_VERSION}</version>
    </dependency>
</dependencies>
```

具体使用AuthzClient可查看官方文档。

### Rest接口

有了这些配置，我们基本上就可以创建一个基于spring boot和keycloak的一个rest服务了。

假如我们为keycloak的client创建了新的用户：alice。

第一步我们需要拿到alice的access token，则可以这样操作：

```sh
 export access_token=$(\
    curl -X POST http://localhost:8180/auth/realms/spring-boot-quickstart/protocol/openid-connect/token \
    -H 'Authorization: Basic YXBwLWF1dGh6LXJlc3Qtc3ByaW5nYm9vdDpzZWNyZXQ=' \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=alice&password=alice&grant_type=password' | jq --raw-output '.access_token' \
 )
```

这个命令是直接通过用户名密码的方式去keycloak服务器中拿取access_token,除了access_token，这个命令还会返回refresh_token和session state的信息。

因为是直接和keycloak进行交换，所以keycloak的directAccessGrantsEnabled一定要设置为true。

上面命令中的Authorization是什么值呢？

这个值是为了防止未授权的client对keycloak服务器的非法访问，所以需要请求客户端提供client-id和对应的client-secret并且以下面的方式进行编码得到的：

```sh
Authorization: basic BASE64(client-id + ':' + client-secret)
```

access_token是JWT格式的，我们可以简单解密一下上面命令得出的token:

```json
    {
 alg: "RS256",
 typ: "JWT",
 kid: "FJ86GcF3jTbNLOco4NvZkUCIUmfYCqoqtOQeMfbhNlE"
}.
{
 exp: 1603614445,
 iat: 1603614145,
 jti: "b69c784d-5b2d-46ad-9f8d-46214add7afb",
 iss: "http://localhost:8180/auth/realms/spring-boot-quickstart",
 sub: "e6606d93-99f6-4829-ba99-1329be604159",
 typ: "Bearer",
 azp: "app-authz-springboot",
 session_state: "bdc33764-fd1a-400e-9fe0-90a82f4873c1",
 acr: "1",
 allowed-origins: [
  "http://localhost:8080"
 ],
 realm_access: {
  roles: [
   "user"
  ]
 },
 scope: "email profile",
 email_verified: false,
 preferred_username: "alice"
}.
[signature]
```

有了access_token,我们就可以根据access_token去做很多事情了。

比如：访问受限的资源：

```sh
curl http://localhost:8080/api/resourcea \
  -H "Authorization: Bearer "$access_token
```

这里的api/resourcea只是我们本地spring boot应用中一个非常简单的请求资源链接，一切的权限校验工作都会被keycloak拦截，我们看下这个api的实现：

```java
 @RequestMapping(value = "/api/resourcea", method = RequestMethod.GET)
public String handleResourceA() {
        return createResponse();
    }
private String createResponse() {
        return "Access Granted";
    }
```

可以看到这个只是一个简单的txt返回，但是因为有keycloak的加持，就变成了一个带权限的资源调用。

上面的access_token解析过后，我们可以看到里面是没有包含权限信息的，我们可以使用access_token来交换一个特殊的RPT的token，这个token里面包含用户的权限信息：

```sh
curl -X POST \
 http://localhost:8180/auth/realms/spring-boot-quickstart/protocol/openid-connect/token \
 -H "Authorization: Bearer "$access_token \
 --data "grant_type=urn:ietf:params:oauth:grant-type:uma-ticket" \
 --data "audience=app-authz-rest-springboot" \
  --data "permission=Default Resource" | jq --raw-output '.access_token'
```

将得出的结果解密之后，看下里面的内容：

```json
    {
 alg: "RS256",
 typ: "JWT",
 kid: "FJ86GcF3jTbNLOco4NvZkUCIUmfYCqoqtOQeMfbhNlE"
}.
{
 exp: 1603614507,
 iat: 1603614207,
 jti: "93e42d9b-4bc6-486a-a650-b912185c35db",
 iss: "http://localhost:8180/auth/realms/spring-boot-quickstart",
 aud: "app-authz-springboot",
 sub: "e6606d93-99f6-4829-ba99-1329be604159",
 typ: "Bearer",
 azp: "app-authz-springboot",
 session_state: "bdc33764-fd1a-400e-9fe0-90a82f4873c1",
 acr: "1",
 allowed-origins: [
  "http://localhost:8080"
 ],
 realm_access: {
  roles: [
   "user"
  ]
 },
 authorization: {
  permissions: [
   {
rsid: "e26d5d63-5976-4959-8683-94b7d85318e7",
rsname: "Default Resource"
}
  ]
 },
 scope: "email profile",
 email_verified: false,
 preferred_username: "alice"
}.
[signature]
```

可以看到，这个RPT和之前的access_token的区别是这个里面包含了authorization信息。

我们可以将这个RPT的token和之前的access_token一样使用。



## Jetty+Jersey框架接入Keycloak

我们有一个老系统，用的embeded Jetty+Jersey，虽然官方提供了Jetty 9.x Adapters，但这是针对standalone而言的，现在几乎没人这么用了，所以还是得自己来。官方有Java Servlet Filter Adapter的教程，但是用的是web.xml的例子，而且语焉不详，所以这里就我自己的摸索提供一点参考。

### Jetty整合Jersey框架

先来看一下Jetty+Jersey的原生例子，涉及两个文件
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

### 整合keycloak
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



### 获取用户权限信息

这块不再举例，自己写个Filter去KeycloakSecurityContext里拿就可以了。



## 参考

https://www.keycloak.org/docs/latest/securing_apps/#_jetty9_adapter

https://stackoverflow.com/questions/22188285/does-embedded-jetty-have-the-ability-to-set-the-init-params-of-a-filter