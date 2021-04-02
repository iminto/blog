---
title: "SuperSet安装"
date: 2021-04-02T11:15:20+08:00
tags: [大数据]
author: baicai
---
Centos8安装Superset.

不建议使用python3.8以下版本。低版本建议使用docker-composer或者helm安装。当然，想要低版本Python安装也不是不可以，其实也就一个conda create 的操作而已。
我是新环境，直接升级就好。
### 初始化，升级到python 3.8（非必须）
```bash
wget https://www.python.org/ftp/python/3.8.1/Python-3.8.1.tgz
yum install gcc gcc-c++ libffi-devel
tar -xvf Python-3.8.1.tgz
cd Python-3.8.1
 ./configure --prefix=/usr/local/python3
make && make install
rm -f /usr/bin/python
rm -f /usr/bin/pip
ln -s /usr/local/python3/bin/python3 /usr/bin/python
ln -s /usr/local/python3/bin/python3 /usr/bin/python
python -V
```
### 安装miniconda

```bash
wget -c https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod 777 Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```
一步一步来，最后把conda的路径加入环境变量。

```bash
conda list #颜值安装是否成功

conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels conda create -n superset python=3.8
conda config --get channels
```
最后进入安装superset环节。创建虚拟环境，这也是低版本python安装高版本python软件的关键。为什么不直接用pip，还是考虑到python版本比较混乱的问题。为什么不用virtualenv，主要是conda能安装一些依赖，减少手动操作。最新版的superset应该是不需要这么麻烦，pip一路到底就可以，不过我还是参考了老的安装教程。有闲的可以直接看官方教程，更简单。

```bash
conda create -n superset python=3.8
source activate superset
pip install apache-superset
```
要退出conda创建的虚拟环节可以用conda deactivate指令。
如果pip太慢，可以加入国内源
```bash
#新建配置文件
touch ~/.pip/pip.conf
#加入如下配置
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn

```
如果出现了类似的报错 
```bash
 gcc: error trying to exec 'cc1plus': execvp: No such file or directory
  error: command 'gcc' failed with exit status 1
  ----------------------------------------
  ERROR: Failed building wheel for python-geohash

```
是因为缺少了 gcc-c++，安装即可。这就是第一步里面的初始化所做的。

如果顺利的话，superset应该是安装成功了。接下来就是官方文档里的初始化操作了。

```bash
superset db upgrade
# Create an admin user (you will be prompted to set a username, first and last name before setting a password)
$ export FLASK_APP=superset
superset fab create-admin

# Load some data to play with
superset load_examples

# Create default roles and permissions
superset init

# To start a development web server on port 8088, use -p to bind to another port
superset run -p 8088 --with-threads --reload --debugger
```
再superset load_examples这一步，可能会卡很久，应该是下载网络数据被GFW卡了，直接中止就好了。
superset默认只绑定localhost，想要外网可访问，可以绑定一个主机。

```bash
#hosts文件里添加一条映射
0.0.0.0 ten
#绑定ten
superset run -h ten  -p 8080 --with-threads --reload
#这样直接绑定IP地址是会报错的
superset run -h 110.6.7.8  -p 8080 --with-threads --reload
```
这操作也有点诡异了些。。
