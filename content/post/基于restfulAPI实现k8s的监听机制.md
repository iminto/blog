---
title: "基于restfulAPI实现k8s的监听机制"
date: 2020-08-11T10:17:04+08:00
archives: "2020"
tags: [k8s]
author: baicai
---

 k8s rest api对rc、svc、ingress、pod、deployment等都提供的watch接口，可以实时的监听应用部署状态。 

在此之前简单先说一下http长连接

## 分块传输编码（Chunked transfer encoding）

超文本传输协议（HTTP）中的一种数据传输机制，允许HTTP由应用服务器发送给客户端应用（ 通常是网页浏览器）的数据可以分成多个部分。分块传输编码只在HTTP协议1.1版本（HTTP/1.1）中提供。
 通常，HTTP应答消息中发送的数据是整个发送的，Content-Length消息头字段表示数据的长度。数据的长度很重要，因为客户端需要知道哪里是应答消息的结束，以及后续应答消息的开始。然而，使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。通常数据块的大小是一致的，但也不总是这种情况。

### Transfer-Encoding

消息首部指明了将 entity 安全传递给用户所采用的编码形式。

Transfer-Encoding 是一个逐跳传输消息首部，即仅应用于两个节点之间的消息传递，而不是所请求的资源本身。一个多节点连接中的每一段都可以应用不同的Transfer-Encoding 值。如果你想要将压缩后的数据应用于整个连接，那么请使用端到端传输消息首部  Content-Encoding 。

当这个消息首部出现在 HEAD 请求的响应中，而这样的响应没有消息体，那么它其实指的是应用在相应的  GET 请求的应答的值。

### 指令

#### chunked

数据以一系列分块的形式进行发送。 Content-Length 首部在这种情况下不被发送。。在每一个分块的开头需要添加当前分块的长度，以十六进制的形式表示，后面紧跟着 '\r\n' ，之后是分块本身，后面也是'\r\n' 。终止块是一个常规的分块，不同之处在于其长度为0。终止块后面是一个挂载（trailer），由一系列（或者为空）的实体消息首部构成。

### 分块编码

分块编码主要应用于如下场景，即要传输大量的数据，但是在请求在没有被处理完之前响应的长度是无法获得的。例如，当需要用从数据库中查询获得的数据生成一个大的HTML表格的时候，或者需要传输大量的图片的时候。一个分块响应形式如下：

```bash
HTTP/1.1 200 OK 
Content-Type: text/plain 
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n 
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n 
\r\n
```

HTTP 1.1引入分块传输编码提供了以下几点好处：

- HTTP分块传输编码允许服务器为动态生成的内容维持HTTP持久连接。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。[动态内容，content-length无法预知]
- 分块传输编码允许服务器在最后发送消息头字段。对于那些头字段值在内容被生成之前无法知道的情形非常重要，例如消息的内容要使用散列进行签名，散列的结果通过HTTP消息头字段进行传输。没有分块传输编码时，服务器必须缓冲内容直到完成后计算头字段的值并在发送内容前发送这些头字段的值。[散列签名，需缓冲完成才能计算]
- HTTP服务器有时使用压缩 （gzip或deflate）以缩短传输花费的时间。分块传输编码可以用来分隔压缩对象的多个部分。在这种情况下，块不是分别压缩的，而是整个负载进行压缩，压缩的输出使用本文描述的方案进行分块传输。在压缩的情形中，分块编码有利于一边进行压缩一边发送数据，而不是先完成压缩过程以得知压缩后数据的大小。[gzip压缩，压缩与传输同时进行]

一般情况HTTP的Header包含Content-Length域来指明报文体的长度。有时候服务生成HTTP回应是无法确定消息大小的，比如大文件的下载，或者后台需要复杂的逻辑才能全部处理页面的请求，这时用需要实时生成消息长度，服务器一般使用chunked编码

## 原理

k8s提供的watch功能是建立在对etcd的watch之上的，当etcd的key-value出现变化时，会通知kube-apiserver，这里的Key-vlaue其实就是k8s资源的持久化。

早期的k8s架构中，kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy，都是直接去watch etcd的，这样就造成etcd的连接数太大（节点成千上万时），对etcd压力太大，浪费资源，因此到了后面，只有kube-apiserver去watch etcd，而kube-apiserver对外提供watch api，也就是kube-controller-manager、kube-scheduler、kubelet、kube-proxy去watch kube-apiserver，这样大大减小了etcd的压力



### Watch API

通过k8s 官网 rest api的描述，可以看到，Watch API实际上一个标准的HTTP GET请求，我们以Pod的Watch API为例

```bash
HTTP Request

GET /api/v1/watch/namespaces/{namespace}/pods
Path Parameters

Parameter   Description
namespace   object name and auth scope, such as for teams and projects
Query Parameters

Parameter   Description
fieldSelector   A selector to restrict the list of returned objects by their fields. Defaults to everything.
labelSelector   A selector to restrict the list of returned objects by their labels. Defaults to everything.
pretty  If ‘true’, then the output is pretty printed.
resourceVersion When specified with a watch call, shows changes that occur after that particular version of a resource. Defaults to changes from the beginning of history. When specified for list: - if unset, then the result is returned from remote storage based on quorum-read flag; - if it’s 0, then we simply return what we currently have in cache, no guarantee; - if set to non zero, then the result is at least as fresh as given rv.
timeoutSeconds  Timeout for the list/watch call.
watch   Watch for changes to the described resources and return them as a stream of add, update, and remove notifications. Specify resourceVersion.
Response

Code    Description
200 WatchEvent  OK
```

从上面可以看出Watch其实就是一个GET请求，和一般请求不同的是，它有一个watch的query parameter，也就是kube-apiserver接到这个请求，当发现query parameter里面包含watch，就知道这是一个Watch API，watch参数默认为true。

==返回值是200和WatchEvent。apiserver首先会返回一个200的状态码，建立长连接，然后不断的返回watch event==

### 源码

原理都讲完了，现在到源码了，很简单。

```java
        OkHttpClient client = makeSSLClient();
        Request request = new Request.Builder()
                .url("https://10.1.1.11:6443/api/v1/watch/namespaces/default/pods")
                .addHeader("Authorization","Bearer "+"eyJhbGciOiBnlQ")
                .get()
                .build();
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                log.info("Watch connection failed. reason: {}", e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (!response.isSuccessful()) {
                    log.info("!response.isSuccessful() {}", response.code());
                }

                try {
                    BufferedSource source = response.body().source();
                    while (!source.exhausted()) {
                        String message = source.readUtf8LineStrict();
                        log.info(message);
                    }
                } catch (Exception e) {
                   log.info("Watch terminated unexpectedly. reason: {}", e.getMessage());
                }

            }
        });
```



剩下的就待自己完善了。

需要注意的是：

1. k8s提供的restful API在此处并不是网上常说的HTTP2协议，它就是HTTP 1.1 长连接
2. 要注意http connection的超时，这里是长连接，超时应该是-1

如果是Java k8s client，可以使用fabric8的watch机制，使用如下：

```java
KubernetesClient client = new DefaultKubernetesClient(config);//使用默认的就足够了
            //由于4.10.2版本有个影响events使用的回归bug，暂时回退到4.9.2版本。详情见官网issue#2328
            //4.10.3已修复
            client.v1().events().inAnyNamespace().watch(new Watcher<Event>() {
                @Override
                public void eventReceived(Action action, Event resource) {
                    log.info("event {} resource:{}" , action.name(),resource.toString());
                    redisService.lSet("k8sevent",resource,3600*24*5);
                }
                @Override
                public void onClose(KubernetesClientException cause) {
                   log.info("Watcher close due to {}" , cause);
                }

            });
```

其本质也是调用的restful API。