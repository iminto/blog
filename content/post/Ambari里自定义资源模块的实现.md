---
title: "Ambari里自定义资源模块的实现"
date: 2020-10-13T09:52:43+08:00
archives: "2020"
tags: [Java]
author: baicai
---

Ambari里主机，集群，用户等等都视为一种资源，对它们的增删改查就是对资源的增删改查。

了解实现Ambari里增加一个资源的流程，就更方便修改Ambari的实现。

### 1.新建控制器层

ambari的控制器层在service包里

```java
package org.apache.ambari.server.api.services.dataspace;

import io.swagger.annotations.Api;
import org.apache.ambari.server.api.resources.ResourceInstance;
import org.apache.ambari.server.api.services.BaseService;
import org.apache.ambari.server.api.services.Request;
import org.apache.ambari.server.controller.spi.Resource;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.HttpHeaders;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.UriInfo;
import java.util.Collections;

@Path("/hdfs/")
@Api(value = "Hdfss", description = "Endpoint for user specific operations")
public class HdfsService extends BaseService {

    @GET
    @Produces("text/plain")
    public Response getUsers(String body, @Context HttpHeaders headers, @Context UriInfo ui) {
        return handleRequest(headers, body, ui, Request.Type.GET, createHdfsResource(null));
    }

    private ResourceInstance createHdfsResource(String path) {
        return createResource(Resource.Type.Hdfs,
                Collections.singletonMap(Resource.Type.Hdfs, path));

    }

}
```

继承BaseService。这里定义访问路径和参数。serivice的方法参数上是看不出VO的类型的。复杂的控制器，可以在一个service里调用另外一个service.



访问参数可以封装成对象，需要新建一个XXXRequest对象，比如

```java
package org.apache.ambari.server.controller;

public class HdfsRequest {

    private Integer id;
    private String path;
    private Integer size;

   //getter,setter略
}
```

这里的xxxRequest是不会像springboot一样自动封装成对象的。

### 2.继承ResourceProvide，实现具体的handleRequest方法

```java
package org.apache.ambari.server.controller.internal;

import com.google.common.collect.ImmutableMap;
import com.google.common.collect.Sets;
import org.apache.ambari.server.controller.AmbariManagementController;
import org.apache.ambari.server.controller.HdfsRequest;
import org.apache.ambari.server.controller.spi.Predicate;
import org.apache.ambari.server.controller.spi.Request;
import org.apache.ambari.server.controller.spi.Resource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashSet;
import java.util.Map;
import java.util.Set;

public class HdfsResourceProvider extends AbstractControllerResourceProvider{
    private static final Logger LOG = LoggerFactory.getLogger(HdfsResourceProvider.class);

    public static final String HDFS_RESOURCE_CATEGORY="Hdfses";

    public static final String HDFS_ID_PROPERTY_ID ="hdfs_id";
    public static final String HDFS_PATH_PROPERTY_ID ="hdfs_path";
    public static final String HDFS_SIZE_PROPERTY_ID ="hdfs_size";

    public static final String HDFS_RESOURCE_HDFS_ID_PROPERTY_ID =HDFS_RESOURCE_CATEGORY+"/"+HDFS_ID_PROPERTY_ID;
    public static final String HDFS_RESOURCE_HDFS_PATH_PROPERTY_ID =HDFS_RESOURCE_CATEGORY+"/"+HDFS_PATH_PROPERTY_ID;
    public static final String HDFS_RESOURCE_HDFS_SIZE_PROPERTY_ID =HDFS_RESOURCE_CATEGORY+"/"+HDFS_SIZE_PROPERTY_ID;

    private static Map<Resource.Type, String> keyPropertyIds = ImmutableMap.<Resource.Type, String>builder()
            .put(Resource.Type.Hdfs, HDFS_RESOURCE_HDFS_ID_PROPERTY_ID)
            .build();

    private static Set<String> propertyIds = Sets.newHashSet(
            HDFS_RESOURCE_HDFS_ID_PROPERTY_ID,
            HDFS_RESOURCE_HDFS_PATH_PROPERTY_ID,
            HDFS_RESOURCE_HDFS_SIZE_PROPERTY_ID
    );

    public static void init(){
        //注入操作在此
    }

     HdfsResourceProvider(AmbariManagementController managementController) {
        super(Resource.Type.Hdfs, propertyIds, keyPropertyIds, managementController);
    }

    @Override
    protected Set<Resource> getResourcesAuthorized(Request request, Predicate predicate){
        Set<String> hdfsIds=getRequestPropertyIds(request,predicate);
        Set<Resource> resources=new HashSet<>();
        //查询数据库略
        for(int i=0;i<3;i++){
            ResourceImpl resource=new ResourceImpl(Resource.Type.Hdfs);
            setResourceProperty(resource,HDFS_RESOURCE_HDFS_ID_PROPERTY_ID,i,hdfsIds);
            setResourceProperty(resource,HDFS_RESOURCE_HDFS_PATH_PROPERTY_ID,"path"+i,hdfsIds);
            setResourceProperty(resource,HDFS_RESOURCE_HDFS_SIZE_PROPERTY_ID,i*100,hdfsIds);
            resources.add(resource);
        }
        return resources;
    }


    private HdfsRequest getRequest(Map<String,Object> properties){
        if(properties==null){
            return new HdfsRequest();
        }
        HdfsRequest hdfsRequest=new HdfsRequest();
        if(properties.get("HDFS_RESOURCE_HDFS_ID_PROPERTY_ID")!=null){
            hdfsRequest.setId(Integer.parseInt(properties.get("HDFS_RESOURCE_HDFS_ID_PROPERTY_ID").toString()));
        }
        if(properties.get("HDFS_RESOURCE_HDFS_SIZE_PROPERTY_ID")!=null){
            hdfsRequest.setSize(Integer.parseInt(properties.get("HDFS_RESOURCE_HDFS_SIZE_PROPERTY_ID").toString()));
        }
        hdfsRequest.setPath(properties.get("HDFS_RESOURCE_HDFS_ID_PROPERTY_ID").toString());
        return hdfsRequest;
    }



    @Override
    protected Set<String> getPKPropertyIds() {
        return new HashSet<>(keyPropertyIds.values());
    }
}

```
这里面定义了各种参数字段和参数完整性校验，对应前端传值，实现CRUD逻辑，调用dao等。ResourceProvider和Request类的作用有交叉。

getRequest()用于从request Map里获取字符串参数组装成对象。



同时需要在AbstractControllerResourceProvider里增加一条记录

```java
 case AlertTarget:
        return resourceProviderFactory.getAlertTargetResourceProvider();
  case ViewInstance:
        return resourceProviderFactory.getViewInstanceResourceProvider();
 case Hdfs:
        return new HdfsResourceProvider(managementController);
default:  throw new IllegalArgumentException("Unknown type " + type);
```



### 3.实现ResourceDefinition

```java
package org.apache.ambari.server.api.resources;

import org.apache.ambari.server.controller.spi.Resource;

import java.util.HashSet;
import java.util.Set;

public class HdfsResourceDefinition extends BaseResourceDefinition {
    {
    }

    /**
     * Constructor.
     *
     * @param resourceType resource type
     */
    public HdfsResourceDefinition() {
        super(Resource.Type.Hdfs);
    }

    /**
     * Obtain the plural name of the resource.
     *
     * @return the plural name of the resource
     */
    @Override
    public String getPluralName() {
        return "hdfses";
    }

    /**
     * Obtain the singular name of the resource.
     *
     * @return the singular name of the resource
     */
    @Override
    public String getSingularName() {
        return "hdfs";
    }

    @Override
    public Set<SubResourceDefinition> getSubResourceDefinitions() {
        final Set<SubResourceDefinition> subResourceDefinitions = new HashSet<>();
        subResourceDefinitions.add(new SubResourceDefinition(Resource.Type.Hdfs));
        return subResourceDefinitions;
    }
}
```

这里跟权限应该就有了关系。

修改ResourceInstanceFactoryImpl，加入自己定义的新类型

```java
case RemoteCluster:
        resourceDefinition = new RemoteClusterResourceDefinition();
        break;

 case Hdfs:
        resourceDefinition = new HdfsResourceDefinition();
        break;

 default:
        throw new IllegalArgumentException("Unsupported resource type: " + type);
    }
```

spi包下Resource接口新增一个枚举

```java
package org.apache.ambari.server.controller.spi;
public interface Resource {
    enum InternalType {
    Cluster,
    Service,
    Setting,
```

### 4.数据库相关

orm包下添加对应的实体类和Dao实现，resource/META-INF下需要手动添加实体类对象全名，比如

```xml
  <persistence-unit name="ambari-server" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>org.apache.ambari.server.orm.entities.AlertCurrentEntity</class>
    <class>org.apache.ambari.server.orm.entities.AlertDefinitionEntity</class>
   </persistence-unit>   
```

### 5.注入相关

AmbariServer类也需要相应注入类依赖的对象


### 6.postman模拟验证

请求

```bash
curl --location --request GET 'http://10.180.210.146:8080/api/v1/hdfs?fields=Hdfses/*' \
--header 'Content-Typ: application/x-www-form-urlencoded; charset=UTF-8' \
--header 'Cookie: AMBARISESSIONID=node0qm8s4v2muk61199wsf0jqgmg1.node0' \
--header 'X-Requested-By: X-Requested-By' \
--data-raw ''
```

返回

```json
{
  "href" : "http://10.180.210.146:8080/api/v1/hdfs?fields=Hdfses/*",
  "items" : [
    {
      "href" : "http://10.180.210.146:8080/api/v1/hdfs/0",
      "Hdfses" : {
        "hdfs_id" : 0,
        "hdfs_path" : "path0",
        "hdfs_size" : 0
      }
    },
    {
      "href" : "http://10.180.210.146:8080/api/v1/hdfs/1",
      "Hdfses" : {
        "hdfs_id" : 1,
        "hdfs_path" : "path1",
        "hdfs_size" : 100
      }
    },
    {
      "href" : "http://10.180.210.146:8080/api/v1/hdfs/2",
      "Hdfses" : {
        "hdfs_id" : 2,
        "hdfs_path" : "path2",
        "hdfs_size" : 200
      }
    }
  ]
}
```

请求必须带上?fields=Hdfses/*,表示展示所有字段，否则查询结果是显示不完整的。



### 7.自由风格Controller

也可以抛开ambari的规则，自由使用javax.ws风格。

但这样就没法使用Ambari内置的权限和谓词风格URL查询了