---
title: "进程Socket描述符的那些事"
date: 2021-07-09T13:52:40+08:00
tags: [Linux]
---

前几天看到有人发的一个面试题，问的是MySQL连接的进程描述符的问题。

在Linux里，一切皆文件，那进程描述符，实际就是文件描述符了。

我们还知道Linux 内核提供了一种通过 proc文件系统，/proc 文件系统是一个虚拟文件系统，通过它可以使用一种新的方法在 Linux内核空间和用户间之间进行通信。在 /proc文件系统中，我们可以将对虚拟文件的读写作为与内核中实体进行通信的一种手段，但是与普通文件不同的是，这些虚拟文件的内容都是动态创建的。用户和应用程序可以通过proc得到系统的信息，并可以改变内核的某些参数。

/proc目录通常对用户来说是只读的，如果你直接在bash下想要修改一个文件是权限不足的。但是对系统来说是可写的，因此也就可以通过编程来实现增删改查。

### 查看socket描述符
那么，这个文件描述符就一定是在/proc 目录了。想必那就是在相应进程的/proc/$pid/fd 目录下存放了此进程所有打开的fd。

```bash
[root@localhost fd]# pwd
/proc/1723/fd
[root@manager 1723]# ll fd|grep socket
lrwx------ 1 root root 64 Jul  7 13:49 103 -> socket:[5722374]
lrwx------ 1 root root 64 Jul  7 13:49 104 -> socket:[5057632]
lrwx------ 1 root root 64 Jul  7 13:49 105 -> socket:[5722375]
lrwx------ 1 root root 64 Jul  7 13:49 106 -> socket:[5057636]
lrwx------ 1 root root 64 Jul  7 13:49 107 -> socket:[5983188]
lrwx------ 1 root root 64 Jul  7 13:49 124 -> socket:[5983189]
lrwx------ 1 root root 64 Jul  7 13:49 130 -> socket:[27456]
lrwx------ 1 root root 64 Jul  7 13:49 131 -> socket:[27458]
lrwx------ 1 root root 64 Jul  7 13:49 132 -> socket:[27460]
lrwx------ 1 root root 64 Jul  7 13:49 51 -> socket:[23447]
lrwx------ 1 root root 64 Jul  7 13:49 52 -> socket:[23448]
lrwx------ 1 root root 64 Jul  7 13:49 78 -> socket:[5057630]
lrwx------ 1 root root 64 Jul  7 13:49 79 -> socket:[5721339]
lrwx------ 1 root root 64 Jul  7 13:49 80 -> socket:[3639663]
lrwx------ 1 root root 64 Jul  7 13:49 81 -> socket:[5057631]
lrwx------ 1 root root 64 Jul  7 13:49 82 -> socket:[5721340]
lrwx------ 1 root root 64 Jul  7 13:49 95 -> socket:[5722372]

```
这个结果，和netant看到的相差无几
```bash
[root@manager ~]# netstat -antp | grep 1723 | wc -l
16
```
当然，这和用lsof统计到的结果应该也是差不多的。之所以说差不多，而不是一样，是因为虽然netstat和lsof虽然也是读取的/proc文件系统，但是有自己的过滤和判断条件，比如这两个工具除了读取/proc/pid/fd目录，还会读取/proc/net/tcp(udp)文件。因此，如果socket创建了，没有被使用，那么就只会在/proc/pid/fd下面有，而不会在/proc/net/tcp(udp)，那么netstat就统计不到了。

那么这个socket:后面的一串数字是什么呢？看起来像是端口号，有些又明显不是，其实是该socket的inode号。
那么，知道了某个进程打开的socket的inode号后，我们可以做什么呢？这就涉及到/proc/net/tcp(udp对应/proc/net/udp)文件了，其中也列出了相应socket的inode号通过比对此字段，我们能在/proc/net/tcp下获得此套接口的其他信息，如对应的<本地地址：端口号，远端地址：端口号>四元组，窗口大小，状态等信息。具体字段含义详见net/ipv4/tcp_ipv4.c 中的 tcp4_seq_show 函数。
```bash
 [root@manager net]# cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 00000000:006F 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 15528 1 ffff880426f60000 100 0 0 10 0
   1: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 19300 1 ffff880426f607c0 100 0 0 10 0
   2: 0100007F:0019 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 20170 1 ffff88042dfc8000 100 0 0 10 0
   3: 00000000:21DE 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 21513 1 ffff88042dfc87c0 100 0 0 10 0
   4: 55F9B40A:0016 59CDB40A:2779 01 00000000:00000000 02:000936FD 00000000     0        0 6004930 2 ffff88042dfca6c0 22 6 1 10 -1
   5: 55F9B40A:0016 59CDB40A:2733 01 00000030:00000000 01:00000018 00000000     0        0 5984362 4 ffff880426f645c0 25 4 31 10 -1
   6: 55F9B40A:0016 59CDB40A:2778 01 00000000:00000000 02:000936FD 00000000     0        0 6004866 2 ffff88042dfcae80 24 7 1 10 -1
   7: 55F9B40A:8F2E 55F9B40A:20F9 01 00000000:00000000 00:00000000 00000000     0        0 22051 1 ffff88042dfc9f00 20 4 30 10 -1
   8: 55F9B40A:0016 59CDB40A:2738 01 00000000:00000000 02:000843C9 00000000     0        0 5983909 2 ffff880426f664c0 22 4 21 7 6
[root@manager net]#
```
这个文件怎么解读呢，我们暂时只看第一部分
```bash
   8: 55F9B40A:0016 59CDB40A:2738 01 
   |      |      |      |      |   |--> connection state（套接字状态）
   |      |      |      |      |------> remote TCP port number（远端端口，主机字节序）
   |      |      |      |-------------> remote IPv4 address（远端IP，网络字节序）
   |      |      |--------------------> local TCP port number（本地端口，主机字节序）
   |      |---------------------------> local IPv4 address（本地IP，网络字节序）
   |----------------------------------> number of entry

```
比如我们看到59CDB40A:2738这个rem_address，很自然它就是TCP的四元组，十六进制转为二进制后就是
89.205.180.10:10040，注意此处IP地址应该是10.180.205.89。用lsof验证下
```bash
[root@manager net]# lsof -i|grep 10040
sshd    17757   root    3u  IPv4 5983909      0t0  TCP manager.bigdata:ssh->10.180.205.89:10040 (ESTABLISHED)
```
connection state(套接字状态)，不同的数值代表不同的状态，参照如下：

    TCP_ESTABLISHED:1   TCP_SYN_SENT:2
    TCP_SYN_RECV:3      TCP_FIN_WAIT1:4
    TCP_FIN_WAIT2:5     TCP_TIME_WAIT:6
    TCP_CLOSE:7         TCP_CLOSE_WAIT:8
    TCP_LAST_ACL:9      TCP_LISTEN:10
    TCP_CLOSING:11


我们看的这条数据并不是MySQL的连接。问题来了，为什么MySQL里看到那么多fd，netstat也看到了很多，但是 /proc/net/tcp 下并没有那么多socket描述符呢。前面说过了，/proc/net/tcp(udp)可以认为是proc/pid/fd的子集，但是这也差的太离谱了。其实原因很简单，如果你在 /proc/net/tcp下找不到，试试去/proc/net/tcp6 下找找呢。

### 关闭指定socket连接
如果我们再进一步，我们现在可以找到每个pid下的socket描述符，如果我想断掉这个描述符也就是断开这个连接，怎么做呢？用防火墙显然不是好主意，防火墙通常是针对某个IP和固定端口的。这个时候，socket fd号就派上用场了。注意，fd和inode是两码事。

查看当前fd
```bash
(base) [root@manager ~]# ll /proc/1723/fd|grep socket
lrwx------ 1 root root 64 Jul  7 13:49 103 -> socket:[10232948]
lrwx------ 1 root root 64 Jul  7 13:49 105 -> socket:[9490029]
lrwx------ 1 root root 64 Jul  7 13:49 107 -> socket:[10232952]
lrwx------ 1 root root 64 Jul  7 13:49 124 -> socket:[10232954]
lrwx------ 1 root root 64 Jul  7 13:49 130 -> socket:[27456]
lrwx------ 1 root root 64 Jul  7 13:49 131 -> socket:[27458]
lrwx------ 1 root root 64 Jul  7 13:49 132 -> socket:[27460]
lrwx------ 1 root root 64 Jul  7 13:49 51 -> socket:[23447]
lrwx------ 1 root root 64 Jul  7 13:49 52 -> socket:[23448]
lrwx------ 1 root root 64 Jul  7 13:49 79 -> socket:[10882498]
```
现在在另外一台服务器，直接用命令行mysql -h连接本机的mysql服务，然后再查看下fd列表
```bash
(base) [root@manager ~]# ll /proc/1723/fd|grep socket
lrwx------ 1 root root 64 Jul  7 13:49 103 -> socket:[10232948]
lrwx------ 1 root root 64 Jul  7 13:49 105 -> socket:[9490029]
lrwx------ 1 root root 64 Jul  7 13:49 107 -> socket:[10232952]
lrwx------ 1 root root 64 Jul  7 13:49 124 -> socket:[10232954]
lrwx------ 1 root root 64 Jul  7 13:49 130 -> socket:[27456]
lrwx------ 1 root root 64 Jul  7 13:49 131 -> socket:[27458]
lrwx------ 1 root root 64 Jul  7 13:49 132 -> socket:[27460]
lrwx------ 1 root root 64 Jul  7 13:49 51 -> socket:[23447]
lrwx------ 1 root root 64 Jul  7 13:49 52 -> socket:[23448]
lrwx------ 1 root root 64 Jul  7 13:49 79 -> socket:[10882498]
lrwx------ 1 root root 64 Jul  7 13:49 80 -> socket:[11222776]
```

对比下，最下面多出来的一行就是新增的那个连接，fd=80,socket inode=11222776。

我们使用gdb调用syscall，关闭这个fd

```bash
gdb -p 1723
call close(80)
quit
```
然后看一下，远程mysql连接已经断了。