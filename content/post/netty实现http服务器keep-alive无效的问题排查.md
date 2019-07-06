---
title: "Netty实现http服务器keep Alive无效的问题排查"
date: 2019-07-06T22:11:37+08:00
tags: [Java]
---

#### netty实现http服务器keep-alive无效的问题排查

今天在用netty实现一个http服务器的时候，发现keep-alive并没有生效，具体表现是在request和response的header里都能看到keep-alive的标志：

```html
request:
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: max-age=0
Connection: keep-alive

response:
HTTP/1.1 200 OK
content-type: text/html;charset=UTF-8
content-length: 0
connection: keep-alive
```

可以看出，无论是请求还是响应，都是keep-alive的，但是请求两次，服务器端日志如下：

```
七月 06, 2019 9:51:27 下午 io.netty.handler.logging.LoggingHandler channelRead
信息: [id: 0xade39344, L:/0:0:0:0:0:0:0:0:8080] READ: [id: 0x26d40041, L:/0:0:0:0:0:0:0:1:8080 - R:/0:0:0:0:0:0:0:1:33130]
七月 06, 2019 9:51:27 下午 io.netty.handler.logging.LoggingHandler channelReadComplete
信息: [id: 0xade39344, L:/0:0:0:0:0:0:0:0:8080] READ COMPLETE
keepAlive=true
channel id=26d40041
http uri: /a.txt?name=chen&f=123;key=456
name=chen
f=123
key=456
七月 06, 2019 9:51:29 下午 io.netty.handler.logging.LoggingHandler channelRead
信息: [id: 0xade39344, L:/0:0:0:0:0:0:0:0:8080] READ: [id: 0x600995e6, L:/0:0:0:0:0:0:0:1:8080 - R:/0:0:0:0:0:0:0:1:33156]
七月 06, 2019 9:51:29 下午 io.netty.handler.logging.LoggingHandler channelReadComplete
信息: [id: 0xade39344, L:/0:0:0:0:0:0:0:0:8080] READ COMPLETE
keepAlive=true
channel id=600995e6
http uri: /a.txt?name=chen&f=123;key=456
name=chen
f=123
key=456
```

客户端两次连接的socket端口一次是33130,第二次是33156，channel id也不一样，证明确实是两个连接，keep-alive并没有生效。

其中，channel id来自这里

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
     //
     System.out.println("channel id="+ctx.channel().id());
}
```

为什么呢，看下代码中Server和ServerHandle也没什么问题，关键代码如下：

```java
serverBootstrap.channel(NioServerSocketChannel.class)
                    .group(boss, work)
                    .handler(new LoggingHandler(LogLevel.INFO))                    // handler在初始化时就会执行，可以设置打印日志级别
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast("http-coder",new HttpServerCodec());
                            ch.pipeline().addLast("aggregator",new HttpObjectAggregator(1024*1024));  //在处理 POST消息体时需要加上
                            ch.pipeline().addLast(new HttpServerHandler());
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childOption(ChannelOption.TCP_NODELAY, true);
//handle代码
httpResponse.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/html;charset=UTF-8");
            httpResponse.headers().setInt(HttpHeaderNames.CONTENT_LENGTH, httpResponse.content().readableBytes());
            if (keepAlive) {
                httpResponse.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
                ctx.writeAndFlush(httpResponse);
            } else {
                ctx.writeAndFlush(httpResponse).addListener(ChannelFutureListener.CLOSE);
            }
```

代码很明显，如果请求是 keep-alive的，那么响应头也加上keep-alive标志，从而实现了长连接。看了半天代码没看出端倪来，突然注意到了在浏览器中，F12看到了/favicon.ico的请求一直是pending的，也就是阻塞在了这里，代码里是这么写的

```java
//去除浏览器"/favicon.ico"的干扰
if (uri.equals(FAVICON_ICO)) {
      return ;
 }
```

这段代码来自我参考的别人的代码，本意是要忽略 /favicon.ico这种无效的请求，但是由于直接return了，导致当前连接被阻塞了，如果这时刷新，就会导致新开一个连接来处理请求。要解决这个问题很简单，只需要注释掉这段代码，对于/favicon.ico请求，直接返回空的200状态码就好了。

现在再来看下请求日志：

```
信息: [id: 0xee8bc5e1, L:/0:0:0:0:0:0:0:0:8080] READ: [id: 0x734e2ebb, L:/0:0:0:0:0:0:0:1:8080 - R:/0:0:0:0:0:0:0:1:37386]
七月 06, 2019 10:03:48 下午 io.netty.handler.logging.LoggingHandler channelReadComplete
信息: [id: 0xee8bc5e1, L:/0:0:0:0:0:0:0:0:8080] READ COMPLETE
keepAlive=true
channel id=734e2ebb
http uri: /a.txt?name=chen&f=123;key=456
name=chen
f=123
key=456
keepAlive=true
channel id=734e2ebb
http uri: /favicon.ico
keepAlive=true
channel id=734e2ebb
http uri: /a.txt?name=chen&f=123;key=456
name=chen
f=123
key=456
keepAlive=true
channel id=734e2ebb
http uri: /favicon.ico
```

无论刷新多少次，服务器端日志里也只记录了一次socket连接日志，并且每次的channel id都是一样的。

顺便再测试下，如果把server中的ChannelOption.SO_KEEPALIVE设置为false，是不会影响http的keep-alive的。
