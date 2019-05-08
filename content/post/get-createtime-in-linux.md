---
title: Linux 下获取文件创建时间
date: 2018-10-07 13:23:59
tags: [Linux]
---

在Linux里，用户层面并没有文件创建时间的概念，无论是用ls还是stat 指令，都无法获取到文件的创建时间
```bash
[tudou@tudou-pc statx]$ stat test-statx.c
  文件：test-statx.c
  大小：6656            块：16         IO 块：4096   普通文件
设备：805h/2053d        Inode：6684737     硬链接：1
权限：(0644/-rw-r--r--)  Uid：( 1000/   tudou)   Gid：( 1001/   tudou)
最近访问：2018-10-07 13:16:29.000000000 +0800
最近更改：2018-10-07 13:21:09.855461986 +0800
最近改动：2018-10-07 13:21:09.855461986 +0800
创建时间：-
```
可以看到「创建时间」一行总是「-」。

如果我们使用百度的话，会看到很多文章说，最近改动时间就是创建时间。的确，我们拿很多文件试验了下，这个最近改动时间（Change Time）确实和创建时间很相近，然而Change time并不是Create time，它实际是文件属性修改时间。
试一下即知：
```bash
[tudou@tudou-pc 下载]$ ./statx ~/.face
statx(/home/tudou/.face) = 0
results=fff
  Size: 7589            Blocks: 16         IO Block: 4096    regular file
Device: 08:05           Inode: 5505043     Links: 1
Access: (0644/-rw-r--r--)  Uid:  1000   Gid:  1001
Access: 2018-09-16 01:15:52.320014139+0800
Modify: 2018-09-16 01:15:52.320014139+0800
Change: 2018-09-16 01:15:52.320014139+0800
 Birth: 2018-09-16 01:15:52.320014139+0800
Attributes: 0000000000000000 (........ ........ ........ ........ ........ ........ ....-... .---.-..)
[tudou@tudou-pc 下载]$ chattr +u ~/.face
[tudou@tudou-pc 下载]$ ./statx ~/.face
statx(/home/tudou/.face) = 0
results=fff
  Size: 7589            Blocks: 16         IO Block: 4096    regular file
Device: 08:05           Inode: 5505043     Links: 1
Access: (0644/-rw-r--r--)  Uid:  1000   Gid:  1001
Access: 2018-09-16 01:15:52.320014139+0800
Modify: 2018-09-16 01:15:52.320014139+0800
Change: 2018-10-07 16:17:10.929769171+0800
 Birth: 2018-09-16 01:15:52.320014139+0800
Attributes: 0000000000000000 (........ ........ ........ ........ ........ ........ ....-... .---.-..)

```
不过，linux也不是完全不支持文件创建时间，文件系统如ext4其实是支持的，只是没有API可以获取到这个数据。比如Java提供的文件API，也就因此无法获取文件创建时间。

不过，自内核 4.11 版本引入的 statx 系统调用支持获取创建时间了，字段名里用的是 btime（Birth time）。

如果用户想要实现在代码里获取这个创建时间，那么只需要调用glibc提供的API即可。但是目前glibc还没有支持，所以只能自己用syscall函数调用。如果仅仅只是想自己实现一个小工具来获取这个时间，那么内核源码树里 samples/statx/test-statx.c 这个文件就是现成的实现。
下载源码：[https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.18.12.tar.xz](https://www.kernel.org/)，选择一个和自己操作系统版本最近的源码分支 .
你要是不想下载几十M的linux源码的话，也可以从[这里](https://elixir.bootlin.com/linux/v4.18.12/source/samples/statx)获取到各个linux版本的源码
编译文件：
```bash
[tudou@tudou-pc statx]$ gcc -O2 -o statx test-statx.c
In file included from /usr/include/sys/stat.h:446,
                 from test-statx.c:28:
/usr/include/bits/statx.h:25:8: 错误：‘struct statx_timestamp’重定义
 struct statx_timestamp
        ^~~~~~~~~~~~~~~
In file included from test-statx.c:26:
/usr/include/linux/stat.h:56:8: 附注：原先在这里定义
 struct statx_timestamp {
        ^~~~~~~~~~~~~~~
In file included from /usr/include/sys/stat.h:446,
                 from test-statx.c:28:
/usr/include/bits/statx.h:36:8: 错误：‘struct statx’重定义
```
注释如下两行代码：
```bash
#define _GNU_SOURCE
#define _ATFILE_SOURCE
```
再次编译即可。
```bash
[tudou@tudou-pc statx]$ gcc -O2 -o statx test-statx.c
[tudou@tudou-pc statx]$ ./statx test-statx.c
statx(test-statx.c) = 0
results=fff
  Size: 6656            Blocks: 16         IO Block: 4096    regular file
Device: 08:05           Inode: 6684737     Links: 1
Access: (0644/-rw-r--r--)  Uid:  1000   Gid:  1001
Access: 2018-10-07 13:16:29.000000000+0800
Modify: 2018-10-07 13:21:09.855461986+0800
Change: 2018-10-07 13:21:09.855461986+0800
 Birth: 2018-10-07 13:16:47.771175840+0800
Attributes: 0000000000000000 (........ ........ ........ ........ ........ ........ ....-... .---.-..)
```
另外一个思路，  使用debugfs来搞。

附：glibc即将支持statx调用[Glibc Support For Statx Is Finally Under Review](https://www.phoronix.com/scan.php?page=news_item&px=Glibc-Statx-Support)

参考： [https://blog.lilydjwg.me/2018/7/11/get-file-birth-time-in-linux.213101.html](https://blog.lilydjwg.me/2018/7/11/get-file-birth-time-in-linux.213101.html)
