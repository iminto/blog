---
title: "Linux恶意ELF文件分析"
date: 2021-05-17T09:31:11+08:00
tags: [Linux]
---

Linux下用来快速分析elf文件有几个工具，一个是readelf，一个是objdump，另外一个是ldd。
通常用ldd来分析动态加载的库，objdump用来反编译。但是这几个工具面对一些恶意文件并不总是有效。
比如对于静态编译的程序，或者变种脚本，ldd就无效了。
对于没有section table的程序，objdump也就可能无法得出结果了。

```bash
#objdump -d ./wi
./wi:     file format elf64-x86-64

```

因为objdump需要依赖code sections或section table，可以用readelf看看

```bash
[root@localhost ~]# readelf -a ./wi
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x5c9e70
  Start of program headers:          64 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         0
  Section header string table index: 0

There are no sections in this file.

There are no sections to group in this file.

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000001ca78b 0x00000000001ca78b  R E    200000
  LOAD           0x0000000000000000 0x00000000005cb000 0x00000000005cb000
                 0x0000000000000000 0x0000000000597d58  RW     1000
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10

There is no dynamic section in this file.

There are no relocations in this file.

The decoding of unwind sections for machine type Advanced Micro Devices X86-64 is not currently supported.

Dynamic symbol information is not available for displaying symbols.

No version information found in this file.
```

所以这种情况下，可以这么来，-D表示对全部文件进行反汇编，-b表示二进制，-m表示指令集架构

```bash
[root@localhost ~]# objdump -b binary -D -m i386 ./wi

./wi:     file format binary
Disassembly of section .data:

00000000 <.data>:
       0:       7f 45                   jg     0x47
       2:       4c                      dec    %esp
       3:       46                      inc    %esi
       4:       02 01                   add    (%ecx),%al
       6:       01 00                   add    %eax,(%eax)
        ...
      10:       02 00                   add    (%eax),%al
      12:       3e 00 01                add    %al,%ds:(%ecx)
      15:       00 00                   add    %al,(%eax)
```

另外，strace，gdb等工具也有一定的帮助。当然，上IDA这种大型武器就更有效了。
这只是浅尝辄止，就算dump出了这么一堆汇编代码，又有什么用，没那么容易看懂。

所以，好端端的一个ELF可执行文件，怎么就没了section table呢？很简单，脚本小子就是喜欢玩些小把戏，很容易就想到是加壳了，Linux下最容易想到的壳就是UPX。

```bash
[root@localhost ~]# strings ./wi |grep UPX
nUPX!(
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.95 Copyright (C) 1996-2018 the UPX Team. All Rights Reserved. $
UPX!u
UPX!
UPX!

```

脚本小子做事还是毛糙，也不晓得隐藏一下壳。把它的王八壳子扒了

```bash
[root@localhost ~]# ./upx -d wi
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   5011080 <-   1878380   37.48%   linux/amd64   wi

Unpacked 1 file.

```

要是脚本小子把UPX壳的信息给隐藏了，那么UPX自带的-d命令就没法脱壳了，这个时候就得用IDA了。加壳的本质就是把原来的程序的数据全部压缩加密了，在静态文件中无法分析，随着程序的执行，运行时会将代码释放到内存中。我们可以用ida远程调试test程序，找到upx自解壳后的 OEP，再把内存给dump出来，就可以实现手动脱壳了。怎样找OEP，这就得看经验了。

脱壳之后呢，继续用strings，strace，netstat等命令做定性分析。

当然了，最简单的就是直接上传到virustotal，最后得出的结果果然是一个Linux.Risk.Bitcoinminer.Tbix。

呵呵，脚本小子。