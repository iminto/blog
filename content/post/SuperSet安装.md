---
title: "SuperSet安装配置"
date: 2021-04-02T11:15:20+08:00
tags: [大数据]
author: baicai
---
Centos8安装Superset。Superset 是 Airbnb （知名在线房屋短租公司）开源的数据探查与可视化平台（曾用名 Panoramix、Caravel ），也就是BI，该工具在可视化、易用性和交互性上非常有特色。

不建议使用python3.8以下版本。低版本建议使用docker-composer或者helm安装。当然，想要低版本Python安装也不是不可以，其实也就一个conda create 的操作而已。我在Centos7+Python 2.7.5 环境下也安装成功。
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
conda list #验证安装是否成功

conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels conda create -n superset python=3.8
conda config --get channels
```

### 使用conda安装SuperSet
最后进入安装superset环节。创建虚拟环境，这也是低版本python安装高版本python软件的关键。为什么不直接用pip，还是考虑到python版本比较混乱的问题。为什么不用virtualenv，主要是conda能安装一些依赖，减少手动操作。最新版的superset应该是不需要这么麻烦，pip一路到底就可以，不过我还是参考了老的安装教程。有闲的可以直接看官方教程，更简单。

```bash
conda create -n superset python=3.8
activate superset
pip install apache-superset
```
要退出conda创建的虚拟环节可以用conda deactivate指令。

查看和切换虚拟环境
```
conda info --envs
conda env list
conda activate base
```
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
### 配置和启动
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
在superset load_examples这一步，可能会卡很久然后失败，原因是example数据是存放在github的，然而某些人没有妈，所以就不能访问了。可以直接中止就好了。或者从github上手动下载回来（压缩包大约28M），然后手动导入（需要起一个HTTP服务，修改Superset源码superset/examples/helpers.py替换BASE_URL，比较麻烦）。

另外superset默认只绑定localhost，想要外网可访问，可以绑定一个主机。

```bash
#hosts文件里添加一条映射
0.0.0.0 ten
#绑定ten
superset run -h ten  -p 8080 --with-threads --reload
#这样直接绑定IP地址是会报错的
superset run -h 110.6.7.8  -p 8080 --with-threads --reload
```
这操作也有点诡异了些。。

> 2021-04-13更新

### 导入样例

前面说过，在superset load_examples这一步会失败，原因是example数据是存放在github的，这个网站被司马佬吃了。这个路径配置在examples/helpers.py里的BASE_URL项。

```bash
#github上下载examples压缩包
wget https://github.com/apache-superset/examples-data/archive/refs/heads/master.zip
#解压，启动本地server
unzip master.zip
python -m SimpleHTTPServer 18089
#PYTHONPATH处理
cp /root/miniconda2/envs/superset/lib/python3.8/site-packages/superset/examples/ /root/py/examples
#修改helpers.py
BASE_URL = "http://10.180.210.146:18089/"
#导入
superset load_examples
```

成功的话，输出应该如下：

```bash
Loaded your LOCAL configuration at [/root/py/superset_config.py]
Loaded your LOCAL configuration at [/root/py/superset_config.py]
logging was configured successfully
INFO:superset.utils.logging_configurator:logging was configured successfully
/root/miniconda2/envs/superset/lib/python3.8/site-packages/flask_caching/__init__.py:201: UserWarning: Flask-Caching: CACHE_TYPE is set to null, caching is effectively disabled.
  warnings.warn(
No PIL installation found
INFO:superset.utils.screenshots:No PIL installation found
Loading examples metadata and related data into examples
Creating default CSS templates
Loading energy related dataset
Creating table [wb_health_population] reference
Loading [World Bank's Health Nutrition and Population Stats]
Creating table [wb_health_population] reference
Creating a World's Health Bank dashboard
Loading [Birth names]
Done loading table!
--------------------------------------------------------------------------------
Creating table [birth_names] reference
Creating some slices
Creating a dashboard
Loading [Random time series data]
Done loading table!
--------------------------------------------------------------------------------
Creating table [random_time_series] reference
Creating a slice
Loading [Random long/lat data]
Done loading table!
--------------------------------------------------------------------------------
Creating table reference
Creating a slice
Loading [Country Map data]
Done loading table!
--------------------------------------------------------------------------------
Creating table reference
Creating a slice
Loading [Multiformat time series]
Done loading table!
--------------------------------------------------------------------------------
Creating table [multiformat_time_series] reference
Creating Heatmap charts
Loading [Paris GeoJson]
Creating table paris_iris_mapping reference
...
```

### 自定义配置及汉化

编辑配置文件后重启即可

```bash
vim /root/miniconda2/envs/superset/lib/python3.8/site-packages/superset/config.py

# Setup default language
BABEL_DEFAULT_LOCALE = "zh"
```

这样直接修改配置文件很麻烦（路径太深），可以把自定义配置文件放到指定位置：

To configure your application, you need to create a file `superset_config.py` and add it to your `PYTHONPATH`. 

All the parameters and default values defined in https://github.com/apache/superset/blob/master/superset/config.py can be altered in your local `superset_config.py`. 



### API文档

API 文档地址：http://10.180.210.146:18088/swagger/v1

虽然SuperSet支持Oauth，但由于SuperSet用的是FlaskAB框架（不是Flask框架），支持比较弱，需要自己写python代码。如果是外部系统对接Superset，只能用cookie登陆的方式调用。

