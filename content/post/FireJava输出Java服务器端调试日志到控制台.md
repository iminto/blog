---
title: FireJava输出Java服务器端调试日志到控制台
date: 2018/01/10 20:21:52
tags: [Java]
---

  针对最新火狐浏览器50+以上版本的firebug协议，类似FirePHP，但是FirePHP已经很久不更新，并且对最新的浏览器也已失效。
> 这个在Firebug之上运行的扩展，结合一个服务器端的库，就可以让你的PHP代码向浏览器发送调试信息，该信息以HTTP响应头（HTTP headers）的方式编码。经过设置，你可以像在Firebug控制台调试JavaScript代码一样得到PHP脚本的警告和错误提示。下面我们来看看具体步骤。

直接上代码
```java
import com.alibaba.fastjson.JSON;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
/**
 * @version V1.0
 * @Description:直接输出服务器端调试日志到控制台，简易版本。
 * @date 2017/6/13 16:51
 */
public class DebugTool {
    public final String VERSION = "2.0.j1";
    public final String HEADER_NAME = "X-ChromeLogger-Data";

    protected Map<String, Object> console = new HashMap<>();
    private String response="";

    public DebugTool() {
        console.put("version", VERSION);
        console.put("columns", new String[]{"log", "backtrace", "type"});
        console.put("rows", new ArrayList<Objects>());
        console.put("request_uri", this.getClass().getName());
    }

    public DebugTool(Class cls) {
        this();
        console.put("request_uri", cls.getName());
    }

    public void log(Object o) {
        log(o,"");
    }

    public void info(Object o) {
        log(o,"info");
    }

    public void warn(Object o) {
        log(o,"warn");
    }

    public void error(Object o) {
        log(o,"error");
    }

    public void log(Object o,String type) {
        Object[] info;
        if(o instanceof Map){
            info = new Object[]{o};
        }else {
            info = new Object[]{o.toString()};
        }
        Object[] obj = new Object[]{info, console.get("request_uri"), type};
        ArrayList<Object> rows = (ArrayList<Object>) console.get("rows");
        rows.add(obj);
        console.put("rows", rows);
    }

    public String getResponse(){
        String json = JSON.toJSONString(console);
        json = Base64.encodeToString(json);
        return json;
    }
}
```
使用方法：
```java
DebugTool debug=new DebugTool(this.getClass());
tool.log("hello 八阿哥");
Map hash=new HashMap();
hash.put("25","张三");
hash.put("19","李四");
tool.warn(hash);
response.add(DebugTool.HEADER_NAME,tool.response);
```
仅对最新版Firefox有效。新版chrome有自己的debug协议（使用websocket）。有趣的是，这本来是一个chrome浏览器支持的协议，后来chrome放弃了，而Firefox拿过来了。
参考：[https://craig.is/writing/chrome-logger](https://craig.is/writing/chrome-logger)
