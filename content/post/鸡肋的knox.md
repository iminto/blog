---
date: "2025-04-03T13:51:15+08:00"
title: "鸡肋的knox"
archives: "2025"
author: baicai
tags: [大数据]
---

网络上介绍Knox的文章很多，然而仅限于demo，很少有人真正深入使用过，更少见谈Knox缺点的文章。故写此博客
# Konx简介
Apache Knox Gateway是一个应用程序网关，用于与Apache Hadoop部署的REST api和ui进行交互。Knox Gateway为与Apache Hadoop集群的所有REST和HTTP交互提供一个单一的访问点。

Apache Knox 的架构设计简单、高效，主要包括以下几个组件：

- Gateway：网关服务，负责处理来自客户端的请求，通过反向代理的方式，将请求转发到后端的 Hadoop 服务上。
- Topology Manager：拓扑管理器，负责管理集群中各个服务的地址信息，并提供服务发现的功能。
- Admin UI：管理界面，提供了一个友好的 Web 界面，供用户进行集群管理。
- SSO（Single Sign-On）：单点登录服务，负责处理用户的身份验证。

特性：

- 提供了一个统一的访问入口，使得用户可以通过一个 URL 访问整个 Hadoop 集群。
- 支持多种身份验证机制，如 LDAP、PAM 等，保证了用户数据的安全性。
- 提供了丰富的访问控制策略，可以灵活地管理用户对集群资源的访问权限。
- 提供了一个易于使用的管理界面，方便用户进行集群管理。

# 安装使用
```bash
unizp /opt/knox-2.0.0.zip
GATEWAY=/opt/knox-2.0.0/
cd $GATEWAY
#启动ldap
bin/ldap.sh start
#创建密钥，随便输一个密码即可
bin/knoxcli.sh create-master
#启动knox
chmod 777 -R /opt/knox-2.0.0/
useradd knox
su - knox -c "/opt/knox-2.0.0/bin/gateway.sh start"
#关闭
su - knox -c "/opt/knox-2.0.0/bin/gateway.sh stop"
```

默认情况下，knox使用内置的ldap作为用户/密码配置存储。

访问地址：https://10.68.6.80:8443/gateway/manager/admin-ui/

## 配置文件

gateway-site.xml：配置knox的默认端口，网关名，kerberos证书，服务白名单等策略
topologies/xx.xml: knox的topology配置，配置网关的认证方式，代理的服务等

knox预置了部分服务的路由配置信息，主要位于$KNOX_HOME/data/services下面

service/hdfsui/2.7.0/service.xml:配置代理HDFS服务的基本信息，包括代理名字，路由规则，2.7.0是版本号，支持代理多个版本
service/hdfsui/2.7.0/rewrite.xml:路由规则的具体实现，使用类似正则表达式的语法进行重写URL

需自定义服务，可在参考预置配置文件进行配置；

以代理HDFS和YARN的UI为例，新建 topologies/ha.xml

主要配置
```xml
<service>
                <role>NAMENODE</role>
                <url>hdfs://10.68.6.80:8020</url>
            </service>
            <service>
                <role>JOBTRACKER</role>
                <url>rpc://10.68.6.80:8050</url>
            </service>    
            <service>
                <role>WEBHDFS</role>
                <url>http://10.68.6.80:50070/webhdfs</url>
            </service>
            <service>
                <role>RESOURCEMANAGER</role>
                <url>http://10.68.6.80:8088/ws</url>
            </service>    
            <service>
                <role>HDFSUI</role>
                <version>2.7.0</version>
                <url>http://10.68.6.80:50070/</url>
            </service>
            <service>
                <role>YARNUI</role>
                <url>http://10.68.6.80:8088</url>
            </service>
```
knox服务最终的URL地址生成规则为：协议 + 主机名 + 端口 + knox根目录 + topology + 服务

http://hostname:8443/gateway/ha/hdfs

http://hostname:8443/gateway/ha/yarn

# SSO功能

Knox本身作为一个网关，除了代理不同的组件外，还可以对外提供SSO服务。

默认情况下，Knox代理了多个服务，只需要登录其中一个，就能无需登录访问另外的服务。

Knox的SSO功能，除了可以给其代理的服务使用外。也可以给任意外部服务使用，以ambari server为例。

访问Ambari首页，将自动跳转到knox登陆页面。


原理：

用户访问Ambari页面时，会判断携带JWT的cookies是否存在，如果不存在则跳转knox网关，认证通过后会下发签名过的JWT给Ambari，Ambari完成验签操作，验签成功后实现登录。

局限性：

Knox的SSO功能比较简单，用户名密码集中存储于LDAP中，缺少角色相关功能，虽然可以扩展，但比较麻烦，也缺少UI管理页面。



# Knox实现原理和局限性

rewrite.xml文件定义了路由规则，一个示例

```xml
  <rule dir="IN" name="HDFSUI/hdfs/inbound/namenode/dfs" pattern="*://*:*/**/hdfs/dfshealth.html">
    <rewrite template="{$serviceUrl[HDFSUI]}/dfshealth.html"/>
  </rule>
```

对于http://hostname:8443/gateway/ha/hdfs/dfshealth.html 这个URL访问，将被重定向到 http://10.68.6.80:50070/dfshealth.html

{$serviceUrl[HDFSUI]} 即为topology中的配置

```xml
<rule dir="OUT" name="HDFSUI/content/static" pattern="/static/{**}">
    <rewrite template="{gateway.url}/hdfs/static/{**}"/>
  </rule>
```

匹配返回页面中所有的` /static/xx.js` 字符串，替换为`http://hostname:8443/gateway/ha/hdfs/static/xx.js`

目的是保证链接能正常跳转。

**rewrite的本质是正则替换。**

你甚至可以把输出中的 title 换成另外一个标题。

其基本原理就是Java中的Filter.

Knox会把所有服务中的rewrite.xml转化成一个超大的Filter配置

## 不同服务下的同名链接

这就带来一问题，如果A服务有 /log/ 这个链接，B 服务也有 /log/ 这个链接，在rewrite的OUT规则中替换时无法区分

导致B服务的  /log/ 被替换为 /A/log 这个（因为OUT规则中无法获取和配置服务名）

两种解决方式

- B服务的 /log/ 链接改名为 /logB/，需要修改B服务的源码
- A和B服务里的 /log/ 改为  log/ ，即使用相对链接，并不进行替换，需要修改A和B服务源码

这就导致如果代理了HDFS和HBASE的UI页面，需要一个页面修改源码，否则会导致在HBASE中点击日志页面，会跳到HDFS页面。

YARN UI也有 log页面，但因为使用了相对URL，所以不会被替换，也就不存在指向其他服务的问题。

## 同一个服务下的相似链接

A 服务有两个链接

master1.com:8001/log

master2.com:8001/log/aa.html

理论上应该先替换`/log/aa.html`，再替换 `/log`，但是这个替换顺序是无法保证的，会导致替换 `/log`的时候把`/log/aa.html`替换了。

因为rewrite.xml里不是标准的正则语法，无法精确匹配结尾。

这就导致 Hbase里的Hbase Master页面不能完美处理其中的region url。

## HA服务

Knox官方文档没有对HA配置和原理做相关介绍，以HDFS为例配置如下

```xml
<service>
   <role>HDFSUI</role>
   <version>2.7.0</version>
    <url>http://10.68.6.80:50070/http://10.68.6.81:50070/</url>
</service>
```

HA功能实现在其对应的service.xml中

```
<dispatch classname="org.apache.knox.gateway.dispatch.URLDecodingDispatch" ha-classname="org.apache.knox.gateway.dispatch.URLDecodingDispatch"/>
```

每个服务都可以定义自己的HA实现，或者使用系统默认的实现

HBASE UI的HA实现类

```xml
<dispatch classname="org.apache.knox.gateway.hbase.HBaseUIDispatch" ha-classname="org.apache.knox.gateway.hbase.HBaseUIHaDispatch"/>
```

遇到的问题：

1.HDFS UI的默认HA实现，其默认指向的节点是url中的第二个节点，除非第一个节点下线。

2.HBASE UI中，Hbase master支持多个实例（大于2），配置三个实例将报错，仅能配置1和2个实例。

# 建议
建议不使用Knox，因为实在太痛苦了。
