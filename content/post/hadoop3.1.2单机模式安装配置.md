---
title: "hadoop 3.1.2 单机模式安装配置"
date: 2019-08-25T23:12:21+08:00
archives: "2019"
tags: [Java,大数据]
author: baicai
---
# hadoop 3.1.2 单机模式安装配置

现在搞大数据记录一下，方便查阅。

### 1.安装配置jdk和下载hadoop略。

hadoop 下载地址：http://mirror.bit.edu.cn/apache/hadoop/common/
使用了较新且保守的3.1.2版本

### 2.配置修改

环境变量修改

```bash
export HADOOP_HOME=/soft/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
```
配置etc/hadoop/hadoop-env.sh

```bash
export JAVA_HOME=/soft/java
export HADOOP_HOME=/soft/hadoop
```

配置etc/hadoop/core-site.xml

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:///develop/data/hadoop</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.104:8888</value>
    </property>
</configuration>
```

配置etc/hadoop/hdfs-site.xml

```xml
<configuration>
        <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///develop/data/hadoop/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///develop/data/hadoop/dfs/data</value>
    </property>
    <property>
    <name>dfs.datanode.du.reserved</name>
    <value>1073741824</value>
    <description>Reserved space in bytes per volume..</description>
</property>
</configuration>
```

### 3.配置免密码SSH登录

```bash
ssh-keygen -t rsa
cat ~/ssh/id_rsa.pub>>~/ssh/authorized_keys
#ssh localhost 测试是否成功 
```

### 4.启动测试

```
#格式化
hdfs namenode -format
#启动hdfs
./sbin/start-dfs.sh
#停止hdfs
./sbin/stop-dfs.sh
#验证是否成功
http://localhost:9870/
```

至此，hadoop的单机模式基本安装结束。

简单的验证hadoop命令：

```
hadoop fs -mkdir /test
```

在浏览器中应该可以看到新建的目录了。

### 注意：

1.网上的教程很多是2.x老版本，3.1.0版本后，hdfs的web 50070端口 -> 9870端口了 。

2.如果webHDFS出错，提示"Failed to retrieve data from /webhdfs/v1/?op=LISTSTATUS:Server Error“，也无法透过Web界面上传文件，一般是JDK版本过高引起的，目前hadoop还只支持JDK8版本。如果是JDK9以上版本，可以编辑hadoop-env.sh

```bash
export HADOOP_OPTS="--add-modules java.activation"
```

3.上传文件/创建目录报错 Permission Denied，修改hdfs-site.xml，设定dfs.permissions=false。按照本文的最新配置就不会遇到这个问题。
