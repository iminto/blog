---
title: "K8s部署springboot"
date: 2020-03-06T13:44:14+08:00
archives: "2020"
tags: [k8s,Java]
author: waitfox@qq.com
---
### 1.安装k8s

安装K8S的步骤略去，使用k3s安装会更快捷方便，方便测试环境。

如果使用k3s会有个坑，k3s默认使用container而不是docker作为容器，会导致运行时出现一些问题，后面会详细分析。

安装k3s后，需要按照如下修改 /etc/systemd/system/k3s.service

```bash
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server --docker \
#添加 --docker 参数
```

### 2.springboot镜像准备

pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.2.5.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.github.iminto</groupId>
	<artifactId>bcdemo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>bcdemo</name>
	<description>Demo project for Spring Boot</description>

	<properties>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```

BcdemoApplication.java 启动器代码略。

src\main\java\com\github\iminto\bcdemo\controller\HomeController.java

```java
package com.github.iminto.bcdemo.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HomeController {
    @RequestMapping("/")
    public String home() {
        return "Hello Docker World";
    }
}
```

src\main\resources\application.yaml

```yaml
server:
    port: 9010
```

mvn打包。

Dockerfile文件如下：

```sh
FROM openjdk:8-jdk-alpine
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
COPY bcdemo.jar /opt/app.jar
COPY run.sh /opt/run.sh

EXPOSE 9010
ENTRYPOINT ["/bin/sh", "/opt/run.sh"]

```

run.sh

```bash
#!/bin/bash
# do other things here
java -jar  /opt/app.jar  2>&1 
```

需要注意，docker必须要有一个前台进程，不然运行后会马上退出，所以不能使用下面的命令

```bash
#这么写是错的
java -jar  /opt/app.jar  2>&1 & 
```

docker镜像构建

```bash
docker build . -t bcdemo:1.0
```

测试docker镜像是否正确时，需要用ctrl+p+q终止控制台日志打印，不要用ctrl+c。

### 3.k8s部署

编写yaml文件

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: k8s-springboot-demo
  labels:
    app: k8s-springboot-demo
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: k8s-springboot-demo
  template:
    metadata:
      labels:
        app: k8s-springboot-demo
    spec:
      containers:
        - name: k8s-springboot-demo
          image: bcdemo:1.0
          ports:
            - containerPort: 9010
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: 9010
            initialDelaySeconds: 30
            timeoutSeconds: 30
          imagePullPolicy: IfNotPresent
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---
apiVersion: v1
kind: Service
metadata:
  name: k8s-springboot-demo
  namespace: default
  labels:
    app: k8s-springboot-demo
spec:
  ports:
    - port: 9010
      targetPort: 9010
  selector:
    app: k8s-springboot-demo
  type: NodePort
```

部署

```bash
#部署
kubectl apply -f sp.yaml
#查看运行状态
kubectl get po,svc,deploy -o wide
#删除Deployment
kubectl delete -f sp.yaml
#删除节点
kubectl delete pod k8s-springboot-demo-fc778b44-f4p49
#查看节点详细运行状态，可用于排错
kubectl describe pod  k8s-springboot-demo-fc778b44-f4p49
```

最终部署结果如下：

```bash
[root@chenwork2 project]# kubectl get po,svc,deploy -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
pod/k8s-springboot-demo-fc778b44-zfjnx   1/1     Running   0          16h   10.42.0.11   chenwork2   <none>           <none>

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE   SELECTOR
service/k8s-springboot-demo   NodePort    10.43.35.126   <none>        9010:31622/TCP   16h   app=k8s-springboot-demo
service/kubernetes            ClusterIP   10.43.0.1      <none>        443/TCP          51d   <none>

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS            IMAGES       SELECTOR
deployment.extensions/k8s-springboot-demo   1/1     1            1           16h   k8s-springboot-demo   bcdemo:1.0   app=k8s-springboot-demo

```

验证：

```bash
[root@chenwork2 project]# curl -i -X GET chenwork2:31622/
HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 18
Date: Fri, 06 Mar 2020 01:55:40 GMT

Hello Docker World
```

这样的端口，只能在集群内访问，集群外是无法访问的。

### 4.k3s的问题

如果pod一直是 ContainerCreating 状态，那就是pod没有创建成功，用describe命令看一下，最后几行一般会看到如下报错

```bash
Warning  FailedCreatePodSandBox  91s (x29 over 21m)  kubelet, host123   Failed create pod sandbox: rpc error: code = Unknown desc = failed to get sandbox image "k8s.gcr.io/pause:3.1": failed to pull image "k8s.gcr.io/pause:3.1": failed to pull and unpack image "k8s.gcr.io/pause:3.1": failed to resolve reference "k8s.gcr.io/pause:3.1": failed to do request: Head https://k8s.gcr.io/v2/pause/manifests/3.1: dial tcp 108.177.97.82:443: i/o timeout
```

 原因已经非常清楚了，failed to pull image "k8s.gcr.io/pause:3.1"，镜像拉不到。 

解决方法：

```bash
docker pull mirrorgooglecontainers/pause:3.1
#其实直接把rancher/pause镜像命名成k8s.gcr.io/pause即可
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
```

重启k3s，你会发现pod还是启动不起来，这就是前面说的原因：**k3s默认使用container而不是docker作为容器**

你有了pause镜像，但是k3s不认docker镜像，而是认container容器，所以需要把k3s改成docker运行时。

或者如下操作：

```bash
ctr --version
#需要预先导出pause镜像
ctr images import pause-amd64-3.1.tar
#看看有没有加载进来
ctr images list
#如果加载了，但是名字不匹配，需要打标签
ctr images tag gcr.io/google_containers/pause-amd64:3.1 k8s.gcr.io/pause:3.1
```

### 5.LoadBalancer服务暴露给外部访问

K8S Service 暴露服务类型有三种：ClusterIP、NodePort、LoadBalancer，三种类型分别有不同的应用场景。

- 对内服务发现，可以使用 ClusterIP 方式对内暴露服务，因为存在 Service 重新创建 IP 会更改的情况，所以不建议直接使用分配的 ClusterIP 方式来内部访问，可以使用 K8S DNS 方式解析，DNS 命名规则为：<svc_name>.<namespace_name>.svc.cluster.local，按照该方式可以直接在集群内部访问对应服务。

- 对外服务暴露，可以采用 NodePort、LoadBalancer 方式对外暴露服务，NodePort 方式使用集群固定 IP，但是端口号是指定范围内随机选择的，每次更新 Service 该 Port 就会更改，不太方便，当然也可以指定固定的 NodePort，但是需要自己维护 Port 列表，也不方便。LoadBalancer 方式使用集群固定 IP 和 NodePort，会额外申请申请一个负载均衡器来转发到对应服务，但是需要底层平台支撑。如果使用 Aliyun、GCE 等云平台商，可以使用该种方式，他们底层会提供 LoadBalancer 支持，直接使用非常方便。

以上方式或多或少都会存在一定的局限性，所以建议如果在公有云上运行，可以使用 LoadBalancer、 Ingress 方式对外提供服务，私有云的话，可以使用 Ingress 通过域名解析来对外提供服务。

下面使用LoadBalancer方式。


#### yaml文件修改

修改`k8s-springboot-demo.yaml`

```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: k8s-springboot-demo
  namespace: default
  labels:  
    app: k8s-springboot-demo
spec:  
  ports:  
    - port: 8080  
      targetPort: 8080
  selector:  
    app: k8s-springboot-demo 
  type: LoadBalancer
```

service的type修改为LoadBalancer，然后

```yaml
kubectl apply -f k8s-springboot-demo.yaml
```

#### 命令修改

用命令修改的方式更方便。

```bash
[root@chenwork2 project]# kubectl delete svc k8s-springboot-demo
service "k8s-springboot-demo" deleted
[root@chenwork2 project]# kubectl get po,svc,deploy
NAME                                     READY   STATUS    RESTARTS   AGE
pod/k8s-springboot-demo-fc778b44-zfjnx   1/1     Running   0          16h

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   51d

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/k8s-springboot-demo   1/1     1            1           16h
[root@chenwork2 project]# kubectl expose deploy k8s-springboot-demo --type=LoadBalancer
service/k8s-springboot-demo exposed
[root@chenwork2 project]# kubectl get po,svc,deploy
NAME                                     READY   STATUS    RESTARTS   AGE
pod/k8s-springboot-demo-fc778b44-zfjnx   1/1     Running   0          16h
pod/svclb-k8s-springboot-demo-crfvw      1/1     Running   0          4s

NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
service/k8s-springboot-demo   LoadBalancer   10.43.112.54   10.180.249.73   9010:32219/TCP   4s
service/kubernetes            ClusterIP      10.43.0.1      <none>          443/TCP          51d

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/k8s-springboot-demo   1/1     1            1           16h
```

这样就能在集群外，直接用浏览器访问了。

也可以使用 ClusterIP+kubectl proxy 方式，但不推荐。

### 6.容器部署yaml中传递参数

可以向容器传递参数到脚本执行一些特殊操作，而且这里变成脚本来启动，这样后续构建镜像基本不需要改 Dockerfile 了 

```bash
#!/bin/bash
# do other things here
java -jar $JAVA_OPTS /opt/project/app.jar $1  2>&1
```

上边示例中，我们就注入 $JAVA_OPTS 环境变量，来优化 JVM 参数，还可以传递一个变量，这个变量大家应该就猜到了，就是服务启动加载哪个配置文件参数，例如：--spring.profiles.active=prod 那么，在 Deployment 中就可以通过如下方式配置了：

```yaml
spec:
  containers:
    - name: project-name
      image: registry.docker.com/project/app:v1.0.0
      args: ["--spring.profiles.active=prod"]
      env:
	   - name: JAVA_OPTS
	     value: "-XX:PermSize=512M -XX:MaxPermSize=512M -Xms1024M -Xmx1024M..."
```

### 7.参考

[Spring Boot 项目转容器化 K8S 部署实用经验分享](https://blog.csdn.net/aixiaoyang168/article/details/96740530)

[K8s 集群使用 ConfigMap 优雅加载 Spring Boot 配置文件](https://blog.csdn.net/aixiaoyang168/article/details/90116097)

[Spring Boot应用容器化及Kubernetes部署](http://fly-luck.github.io/2018/11/10/Spring Boot App on Kubernetes/)

[基于Kubernetes和Springboot构建微服务](https://www.jianshu.com/p/a9aec78418df)

[Docker / Kubernetes部署Java / SpringBoot项目](https://blog.csdn.net/hehuanchun0311/article/details/84316194)

[在Kubernetes中部署spring boot应用](https://tomoyadeng.github.io/blog/2018/09/13/springboot-k8s-1/index.html)
