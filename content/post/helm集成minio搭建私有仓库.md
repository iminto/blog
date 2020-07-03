---
title: "Helm集成minio搭建私有仓库"
date: 2020-07-03T10:20:40+08:00
archives: "2020"
tags: [k8s]
author: baicai
---
# helm3集成minio搭建私有仓库

我们一般是从本地的目录结构中的chart去进行部署，如果要集中管理chart,就需要涉及到repository的问题可以通过minio建立一个私有的存放仓库。 

## minio安装

安装过程略去，直接下载执行文件即可

```bash
 .\minio.exe server g:/tmp/
```

配置mc

```bash
mc config host add minio http://192.168.1.51 BKIKJAA5BMMU2RHO6IBB V7f1CwQqAcwo80UEIJEjc5gVQUSSx5ohQ9GSrr12
```

## 仓库创建

```bash
helm  create helm-chart
helm package ./helm-chart --debug
#构建索引
helm repo index ./
```

接下来是复制生成的index.yaml到minio中

```bash
.\mc.exe policy set download minio/data
.\mc.exe cp G:\data\project\helm\index.yaml minio/data/
```

到这一步基本就快好了，然后

```bash
 helm repo add mi http://10.180.204.129:9000/data/
 helm repo list
```

最终可以看到如下结果，说明添加仓库成功：

```bash
PS G:\data\project\helm> helm repo list
NAME    URL
stable  https://kubernetes-charts.storage.googleapis.com/
mi      http://10.180.204.129:9000/data/
PS G:\data\project\helm>
```
搜索下
```bash
PS G:\tmp\data> helm search repo mi/helm-chart
NAME            CHART VERSION   APP VERSION     DESCRIPTION
mi/helm-chart   0.2.0           1.16.4          A Helm chart for Kubernetes
```
也能搜到新加的Chart，完工。

## 注意

1.minio虽然是一个文件对象服务器，但是也支持直接在OS文件系统下的操作。也就是说，直接在文件夹上的操作会同步到minio的数据库中。此前一直顾虑minio这种文件系统是否适合用来做repo，其实是多虑的。

2.我下载的是helm v3版本，但打包出来的却是tgz文件，helm2格式的仓库组织形式。自己添加helm3形式的仓库却总是报错找不到index.yaml文件，
也不清楚为何。但是我看helm3内置了一个stable仓库却是helm3文件夹格式的组织结构，不太清楚该怎么操作

3.如果你发布了两个版本的chart包，也update了，但是仓库默认只能搜到高版本的。需要这么做
```bash
PS G:\tmp\data> helm search repo mi/helm-chart --versions
NAME            CHART VERSION   APP VERSION     DESCRIPTION
mi/helm-chart   0.2.0           1.16.4          A Helm chart for Kubernetes
mi/helm-chart   0.1.0           1.16.0          A Helm chart for Kubernetes
```
安装
```bash
helm install mi/helm-chart --version 0.1.0 
```

