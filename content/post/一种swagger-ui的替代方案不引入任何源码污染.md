---
title: "一种swagger Ui的替代方案不引入任何源码污染"
date: 2020-05-12T17:18:49+08:00
archives: "2020"
tags: [Java]
author: baicai
---

在后端项目中，难免遇到需要写接口文档方便第三方调用的场景，一般业界最常用的方案是使用swagger。Java项目中，一般采用springfox项目，它集成了swagger和swagger-ui，不需要单独部署项目，可让文档随着项目一起发布。

但是开源项目往往是开源一时热，事后拂衣去，缺少维护。这个项目已经两年多没有维护了，很多人在issue反馈过bug，作者一年前表示自己比较忙，没空维护。

springfox最新的版本是2.9.2，不支持spring5（虽然有个快照版支持spring5，但一直没发布，整合也有点麻烦）。spring5比较大的一个改变就是增加了webflux，因此旧版springfox无法兼容spring5的。

其实用快照版，稍作修改也能让springfox支持webflux，但是我不是很喜欢这种做法。一个是增加了打包体积和运行内存占用，另一个则是swagger的使用污染了Java源码，很是不美观，强迫症不能忍。

```java
@RestController
@RequestMapping("/dataspace/api/v1/hive")
@Api(value = "hive", description = "hive资源管理")
public class HiveManagerController {
    @Autowired
    HiveManagerService hiveManagerService;

    @RequestMapping(value = "/list", method = {RequestMethod.POST})
    @ApiOperation(value = "资源列表", notes = "")
    public PageResult<HiveVO> showPublic(@ApiParam(value = "hive查询对象")
                                          @RequestBody PageReqParam<HiveReq> hiveReq) {
        PageResult<HiveVO> result = new PageResult<>();
        if (hiveReq.getReqParam() == null) {
            result.setCode(-1);
            result.setMsg("参数不完整");
            return result;
        }
        if (hiveReq.getPageSize() > 50 || hiveReq.getPageSize() < 0) {
            result.setCode(-1);
            result.setMsg("页码非法");
            return result;
        }
        result = hiveManagerService.getList(hiveReq);
        return result;
    }
```

源码中混入了各种ApiParam、Api、ApiOperation注解。

再加上我现在使用的springcloud套件，需要在gateway的feign接口上加注释，这样的话，无论是springfox，还是很多第三方的api doc工具都很难胜任。

于是，我想到了另外一种方法，就是javadoc。然而javadoc自带的注解很有限，不能满足第三方对文档的需求，比如

```java
    /**
     * 根据节点名删除主机
     * @method DELETE
     * @path host/delHostByNodeName
     * @param nodeName 节点名
     * @param cluster 集群名
     * @return JSON
     */
    @DeleteMapping("/delHostByNodeName")
    public String delHostByNodeName(@RequestParam("nodeName") String nodeName,@RequestParam("cluster") String cluster);

```
javadoc并不认识method和path这两个标签，生成的文档还是缺少一些必须要的信息。

这个不难，扩展下taglet即可。

先引入maven依赖

```xml
<dependency>
            <groupId>jdk.tools</groupId>
            <artifactId>jdk.tools</artifactId>
            <version>1.8</version>
            <scope>system</scope>
            <systemPath>${JAVA_HOME}/lib/tools.jar</systemPath>
        </dependency>
```

扩展taglet代码

```java
package com.github.cloud.ali.common.tool;
import com.sun.javadoc.Tag;
import com.sun.tools.doclets.Taglet;
import java.util.Map;

public class MethodTaglet implements Taglet {
    private String NAME = "HTTP请求类型";
    private String HEADER = "HTTP请求类型:";

    @Override
    public boolean inField() {
        return false;
    }

    @Override
    public boolean inConstructor() {
        return false;
    }

    @Override
    public boolean inMethod() {
        return true;
    }

    @Override
    public boolean inOverview() {
        return true;
    }

    @Override
    public boolean inPackage() {
        return true;
    }

    @Override
    public boolean inType() {
        return true;
    }

    @Override
    public boolean isInlineTag() {
        return false;
    }

    public static void register(Map tagletMap) {
        MethodTaglet tag = new MethodTaglet();
        Taglet t = (Taglet) tagletMap.get(tag.getName());
        if (t != null) {
            tagletMap.remove(tag.getName());
        }
        tagletMap.put(tag.getName(), tag);
    }

    @Override
    public String getName() {
        return NAME;
    }

    @Override
    public String toString(Tag tag) {
        return "<DT><B>" + HEADER + "</B><DD>"
                + "<table cellpadding=2 cellspacing=0><tr><td bgcolor=\"yellow\">"
                + tag.text()
                + "</td></tr></table></DD>\n";
    }

    @Override
    public String toString(Tag[] tags) {
        if (tags.length == 0) {
            return null;
        }
        String result = "\n<DT><B>" + HEADER + "</B><DD>";
        result += "<table cellpadding=2 cellspacing=0><tr><td bgcolor=\"yellow\">";
        for (int i = 0; i < tags.length; i++) {
            if (i > 0) {
                result += ", ";
            }
            result += tags[i].text();
        }
        return result + "</td></tr></table></DD>\n";
    }
}

```
同理，path注解也是类似的实现。编译命令如下：
```bash
javadoc -protected -splitindex -use -author -version -encoding utf-8 -charset utf-8 -d /usr/jackma/doc  -windowtitle "ali 文档"  $(ls /usr/jackma/ali/ali-common/src/main/java/com/github/cloud/ali/common/model/*.java |tr "\n" " ") $(ls /usr/jackma/ali/ali-gateway/src/main/java/com/github/cloud/ali/feign/*.java |tr "\n" " ") -tag method:a:"HTTP请求方法:" -tag path:a:"请求路径:"  -tagletpath /usr/jackma/ali/ali-common/src/main/java/com/github/cloud/ali/common/tool/MethodTaglet.java  -tagletpath /usr/jackma/ali/ali-common/src/main/java/com/github/cloud/ali/common/tool/PathTaglet.java -taglet com.github.cloud.ali.common.tool.MethodTaglet -taglet com.github.cloud.ali.common.tool.PathTaglet
```

最终效果如下：

![ javadoc ]( https://s1.ax1x.com/2020/05/12/YNRadK.png)

还可以进一步，加上数据类型的注解，这样就更完善了。

虽然离swagger-ui还有点差距，但是还是比原版javadoc好多了。最大的优点是没有任何限制和对源码的污染。

不得不说，Java的扩展性不是盖的。