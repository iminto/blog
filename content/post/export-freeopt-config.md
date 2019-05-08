---
title: 导出freeOTP中的配置
date: 2018-05-14 22:06:05
---
之前公司的一个网站使用了OTP来做二次验证，然后我就在手机上安装了freeotp这款软件来管理OTP密码，等到换手机了，才发现没法导出原手机的配置，这就尴尬了。FreeOTP is sponsored and officially published by Red Hat，也算是大家闺秀出品的软件，居然不支持这么重要的功能。

试了很多方法，在手机的文件管理器中到处搜索，都没有找到这个配置，基本可以确定freeotp把密钥存放在了系统目录，没有root的话，是没法查看和处理系统目录下的文件，即使用备份工具也备份不出来。

当初网站的OTP二维码也找不到了，网站也没找到重新设置OTP的入口，本着万事不求人的想法，暂时还不想最后求助运维。看来唯一的办法就是root手机了，试了很多工具，没想到kingroot居然支持root魅蓝手机了。

root成功后，马上去freeotp的配置存储目录找到配置文件，找到 /data/data/org.fedorahosted.freeotp/shared_prefs/tokens.xml 文件，得到如下的配置,配置中的引号被转义了
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="bbcx@qq.com:chen">{"algo":"SHA256","counter":0,"digits":6,"issuerExt":"bbcx@qq.com","label":"chen","period":30,"secret":[17,-56,-42,-70,-48,-79,53],"type":"TOTP"}</string>
    <string name="bbc">{"algo":"SHA1","counter":0,"digits":6,"issuerExt":"","label":"bbc","period":30,"secret":[0,1,2,3],"type":"TOTP"}</string>
    <string name="tokenOrder">["bbcx@qq.com:bbcx","bbc"]</string>
</map>

```

可以看出，这里面是就是关于otp的全部配置了，最关键的就是secret字段，这里做了加密，反复试验了半天，没找到解决方案，最终想到Google，找到了这个解决方案：
[https://github.com/viljoviitanen/freeotp-export/blob/master/README.md](https://github.com/viljoviitanen/freeotp-export/blob/master/README.md)
，只需要把tokens.xml贴到这里，[https://rawgit.com/viljoviitanen/freeotp-export/master/export-xml.html](https://rawgit.com/viljoviitanen/freeotp-export/master/export-xml.html)，就能还原出二维码来，用新手机扫描就好了。

事情还没完，最后想去freeotp的官方那里反应下，没想到官方的态度让我大跌眼镜，[https://github.com/freeotp/freeotp-android/issues/20](https://github.com/freeotp/freeotp-android/issues/20)，“出门右转买收费软件去，老子就是不增加备份功能，你能咋地”。
```
"'''Can I create backupcodes'''?

''No, but if you're using an Android smartphone you can replace the Google Authenticator app with Authenticator Plus.
It's a really nice app that can import your existing settings, sync between devices and backup/restore using your sd-card.
It's not a free app, but it's well worth the money.''"

This proprietary app, Authenticator Plus, does look very nice and has some nice features, but the most beneficial I think is its ability to backup and restore codes.

This could be a huge addition to FreeOTP and I would like to request that someone considers this feature and looks at a way of implementing it. I am not able to code myself.
```


最终，在用户义愤填膺的评论下，发现这个软件 [andOTP](https://github.com/flocke/andOTP)，真是兴奋，满足了我对OTP软件的所有需求，也支持备份和导入，极力推荐。

立马卸载了拽拽的freeOTP，装上andOTP，感觉整个世界都阳光明媚。开源的傲慢真是领悟了，惹不起惹不起。
