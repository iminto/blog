---
title: 折腾阿里云OSS的API
date: 2018/01/25 23:26:44
tags: [Java]
---
  这两天想给博客做个插件,利用阿里云的OSS来存储文件.但阿里的文档和代码都烂的超乎想象,要么代码老旧不堪,要么跟小脚老太一样引入一坨依赖,想必这块是外包团队做的吧,或者阿里非核心业务员的技术水平也就这样吧.

  所以想绕开阿里云官方提供的代码自己整一套OSS的API,先跑一个上传文件的demo,能在客户端跑通后再用代码去实现.最简单的方法就是用[REST client](https://github.com/wiztools/rest-client)来模拟.折腾了一下,还挺费劲,记录下折腾过程

  先来试试上传文件,选择PUT方法,要请求的URL为http://baicaidoc.oss-cn-shenzhen.aliyuncs.com/image/small/mm1.jpg
,添加以下header,header头需要包含哪些内容可以看[这里](https://help.aliyun.com/document_detail/31955.html)
```js
Authorization:OSS
LTAIxkX6Qj2OuMZ6:tLZ7nYYP/hkCJbG/6gkOJ7Mi4E=
Date:Thu, 25 Jan 2018 15:20:39 GMT
Content-Disposition:attachment;filename=ivy.jpg
Host:baicaidoc.oss-cn-shenzhen.aliyuncs.com
Content-Encoding:utf-8
```
然后在body里添加file body.
至于header头怎么写和Authorization字段计算的方法,文档里说的比较清晰了[https://help.aliyun.com/document_detail/31951.html](https://help.aliyun.com/document_detail/31951.html).
  尤其需要注意的是Date必须是GMT格式,这个对Java来说也好办,不过要注意时区的问题,GMT时间比东八区慢了8个小时.还有Host需要带上bucket,这在早期是不需要的(早期带上反而会报错SignatureDoesNotMatch)
  另外就是这个Authorization字段的签名需要注意,base64需要处理byte[]数组,而不是字符串.所以用网上的在线验证工具是验证不了的.
Java版的签名代码如下:
```java
import bai.tool.Base64;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;

/**
 * Hello world!
 *
 */
public class App
{
    public static byte[] hamcsha1(byte[] data, byte[] key)
    {
        try {
            SecretKeySpec signingKey = new SecretKeySpec(key, "HmacSHA1");
            Mac mac = Mac.getInstance("HmacSHA1");
            mac.init(signingKey);
            return mac.doFinal(data);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static void main( String[] args )  {
        String toSign="PUT\n" +
                "\n" +
                "image/jpeg; charset=UTF-8\n" +
                "Thu, 25 Jan 2018 15:20:39 GMT\n" +
                "/baicaidoc/image/small/mm1.jpg";
        String accessKey="OrzrzxIsfpFjA7S7yk0Lwy8Bw21TLhquhboiip56";
        byte[] hm=hamcsha1(toSign.getBytes(),accessKey.getBytes());
        System.out.println("OSS LTAIxkX6Qj2OuMZ6:"+Base64.encodeToString(hm));
    }
}
```
  客户端能跑通就好办了，最后是代码，使用HttpURLConnection来实现PUT上传代码。阿里云的OSS SDK太重了，而一般常用的就上传和删除功能
```java
import java.io.*;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URL;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;
import java.util.TimeZone;

public class OSSUpload {

    public String httpUrlConnectionPut(String fileName) {
        String result = "";
        URL url = null;
        String httpUrl = "http://baicaidoc.oss-cn-shenzhen.aliyuncs.com/image/small/test.jpg";
        try {
            url = new URL(httpUrl);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
        if (url != null) {
            HttpURLConnection urlConn;
           try {
                urlConn = (HttpURLConnection) url.openConnection();
                File file = new File(fileName);
                urlConn.setRequestProperty("content-type", "image/jpeg; charset=UTF-8");
                urlConn.setDoOutput(true);// http正文内，因此需要设为true, 默认情况下是false;
                urlConn.setDoInput(true);// 设置是否从httpUrlConnection读入，默认情况下是true;
                urlConn.setConnectTimeout(15 * 1000);
                urlConn.setRequestProperty("User-Agent", "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36");
                //设置请求方式为 PUT
                urlConn.setRequestMethod("PUT");             urlConn.setRequestProperty("Connection", "Keep-Alive");

                SimpleDateFormat sdf = new SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss 'GMT'", Locale.US);
                sdf.setTimeZone(TimeZone.getTimeZone("GMT"));
                urlConn.setRequestProperty("Host", "baicaidoc.oss-cn-shenzhen.aliyuncs.com");
                urlConn.setRequestProperty("Content-Encoding", "UTF-8");
                urlConn.setRequestProperty("Date", sdf.format(new Date()));
                urlConn.setRequestProperty("Content-Length", String.valueOf(file.length()));
                urlConn.setRequestProperty("Authorization", "OSS LTAIxk223j2OuMZ6:tLZ74YYP/hkCJbG/6gkOJ7Mi4E=");
                DataOutputStream dos = new DataOutputStream(urlConn.getOutputStream());
                //写入请求参数

               try {
                   InputStream in = new FileInputStream(file);
                   int bytes = 0;
                   byte[] bufferOut = new byte[4096];
                   while ((bytes = in.read(bufferOut)) != -1) {
                       dos.write(bufferOut, 0, bytes);
                   }
                   dos.flush();
                   dos.close();
                   InputStream is = urlConn.getInputStream();
                   int ch;
                   StringBuffer b = new StringBuffer();
                   while ((ch = is.read()) != -1) {
                       b.append((char) ch);
                   }
                   System.out.println("result:" + b.toString());
               }catch (IOException e){
                   e.printStackTrace();
                   InputStream is=urlConn.getErrorStream();
                   int ch;
                   StringBuffer b = new StringBuffer();
                   while ((ch = is.read()) != -1) {
                       b.append((char) ch);
                   }
                   System.out.println("error result:"+b.toString());
               }
                urlConn.disconnect();
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

           }
        }

        return result;
    }

    public static void main(String[] args) {
        OSSUpload oss = new OSSUpload();        oss.httpUrlConnectionPut("/home/chen/Desktop/tmp/sd.png");
    }
}
```
