---
title: linux下解压bin文件
date: 2018-06-02 13:08:41
tags: [Linux]
---
现在的一些Linux软件很流行使用bin这种安装包格式，只需要下载个安装包就能自动安装解压，比tar.gz省事，比.deb，.rpm的安装包兼容性强，适应范围广。但也有一个问题，bin安装包让人无法知道里面的细节，还是有所顾虑的。比如我前几天需要下载一个JRE6，但Oracle官方在JDK7之前都没有tar.gz包，只有bin包。我肯定不能直接安装bin文件啊，这会破坏我本机已有的JDK8开发环境。

怎么从bin文件里提取出原始安装包呢？其实很简单。用vi打开一个bin文件就知道了，bin文件其实就是一个sh文件和二进制文件的合并文件，前面一段是sh命令，负责实际的安装，它会提取后半部分的二进制数据，后半部分一般是个压缩文件包或者自解压文件的二进制流。
```bash
vi jre-for-linux.bin
```
可以看到，第一行是
```bash
#!/bin/bash
```
接下来就是一堆安装和设置环境变量，提取解压部分了，最关键的部分在这几行
```bash
outname=install.sfx.$$
tail ${tail_args} +162 "$0">$outname
chmod +x $outname
```
继续往下看，267行是exit 0，从第268行开始，就是一堆看似乱码的二进制了，到这里那就清晰多了
```bash
# 从268行起提取二进制文件
tail -n +268 jre-for-linux.bin >install.sfx
# 因为是sfx格式，就用7z解压
7z x install.sfx
```
到此解压成功。手动安装，使用export设置临时变量，就用上了JRE6了。
