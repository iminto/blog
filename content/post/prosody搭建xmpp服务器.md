---
title: "Prosody搭建xmpp服务器"
date: 2020-08-26T10:22:59+08:00
tags: [Linux,闲扯淡]
author: 白菜
---

按惯例上Prosody 自己的文档: https://prosody.im/doc/

### 安装

使用centos8安装

```bash
yum install prosody
dnf --enablerepo=PowerTools install lua-filesystem
```
其它版本linux则无需单独安装lua-filesystem依赖。

### 配置

主配置文件 prosody.cfg.lua 一般不需要修改。

 下面写些咱做的修改😂 

- 在 modules_enabled 中取消启用 version 和 uptime 模块，顺便启动些其他的模块，比如offline。
- 如果需要允许在客户端上注册的话，把 allow_registration 设置成 true 。

其它配置保持默认即可。

另外一个配置文件就是具体和域名对应的配置文件了，位于/etc/prosody/conf.d目录下
我的配置是：
baidecai.xyz.cfg.lua

```lua
VirtualHost "baidecai.xyz"
http_host = "www.baidecai.xyz"
	-- enabled = false -- Remove this line to enable this host

	-- Prosody will automatically search for a certificate and key
	-- in /etc/prosody/certs/ unless a path is manually specified
	-- in the config file, see https://prosody.im/doc/certificates
	ssl = {
		key = "/etc/prosody/cer/baidecai.xyz.key";
		certificate = "/etc/prosody/cer/baidecai.xyz.crt";
		protocol = "tlsv1_1+";
		--- 为客户端到服务器（c2s）和服务器到服务器（s2s）打开认证
		verify = { "peer", "peer" };
		ciphers = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
		dhparam = "/etc/prosody/certs/dh-1024.pem"
	}
disco_items = {
    { "upload.baidecai.xyz" },
 }
Component "upload.baidecai.xyz" "http_upload"
	http_upload_file_size_limit = 1024000
	http_upload_expire_after = 60 * 60 * 24 * 7
	http_upload_path = "/uploaded/files"

http_files_dir = "/uploaded/files"
```

为了支持聊天中发送文件，我加入了http_upload模块。需要注意的是，这个模块来自社区，并不是prosody自带的，所以需要自己去下载放入prosody的插件目录（在这个问题上，我折腾了好几天才搞定，官方文档没有说清楚），要不然你的xmpp就没法发文件了，及时客户端支持发送操作也会报错。

prosody的插件目录位置可以通过这个命令查看：

```bash
prosodyctl about
```

社区插件下载地址： https://hg.prosody.im/prosody-modules/file/tip 
记得给http_upload_path赋予可写权限
重启即可。

### 注意

1. 现在的xmpp client基本都不再支持非SSL登陆了，所以你必须要有一个证书。也就是前文配置中的certificate和key文件，这个很好申请，推荐网址： https://freessl.cn/ 。
2. dhparam文件生成指令

```bash
openssl dhparam -out dh.pem 1024 
```

3. 如果没开启允许客户端注册的话，用 prosodyctl 注册账户
```bash
prosodyctl adduser <JID>
```
到此为止，你已经拥有了一个可以加密聊天，可以发文件的xmpp server了。