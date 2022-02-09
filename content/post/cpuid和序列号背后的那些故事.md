---
title: "Cpuid和序列号背后的那些故事"
date: 2022-02-09T14:16:58+08:00
tags: [Linux]
---

最近测试反馈了一个问题，每次重启服务器，我们某个版本的业务系统中的机器码都会改变，导致根据机器码算出来的许可证失效，从而使软件无法使用。
这个问题反馈了有一段时间了，但是本地一直没复现。然后前几天测试说又复现了，马上去看了下测试环境，服务器是一台国产化FT S2500服务器,验证了下，果然如此，马上去看了下关键代码。

```java
public static String executeLinuxCmd(int type) {
        try {
            String cmd = "dmidecode |grep 'Serial Number'";
            if (type == 1) {
                cmd = "fdisk -l";
            }
            //...
        } catch (IOException e) {//
        }
        return null;
    }
    
    public static String getSerialNumber(int type, String record, String symbol) {
            String execResult = executeLinuxCmd(type);
            String[] infos = execResult.split("\n");
            for (String info : infos) {
                info = info.trim();
                if (info.indexOf(record) != -1) {
                    String[] sn = info.replace(" ", "").split(symbol);
                    return sn[1];
                }
            }
       //...
        }
        
      /**
     * 获取CPUID、硬盘序列号、MAC地址、主板序列号
     *
     * @return
     */
    public static Map<String, String> getAllSn() {
        String os = System.getProperty("os.name");
        Map<String, String> snVo = new HashMap();
        if ("LINUX".equalsIgnoreCase(os)) {
            String mainboardNumber = getSerialNumber(0, "Serial Number", ":");
            String diskNumber = getSerialNumber(1, "Disk identifier", ":");
            snVo.put("diskid", diskNumber == null ? "tmpDiskId" : diskNumber.toUpperCase().replace(" ", ""));
            snVo.put("mainboard", mainboardNumber == null ? "tmpMainboard" : mainboardNumber.toUpperCase().replace(" ", ""));
        } else {
```
这下明白了，它是取的CPU序列号作为机器码。dmidecode的输出中有多个Serial Number，它只取了第一个，恰恰就是Processor Information，也就是我们常说的CPU序列号。
```bash
Handle 0x0001, DMI type 4, 48 bytes
Processor Information
        Socket Designation: CPU0
        Type: Central Processor
        Family: ARMv8
        Manufacturer: Phytium
        ID: 33 66 1F 70 00 00 00 00
        Signature: Implementor 0x70, Variant 0x1, Architecture 15, Part 0x663, Revision 3
        Version: S2500
        Voltage: 0.8 V
        External Clock: 100 MHz
        Max Speed: 2100 MHz
        Current Speed: 2100 MHz
        Status: Populated, Enabled
        Upgrade: Unknown
        L1 Cache Handle: 0x1001
        L2 Cache Handle: 0x1002
        L3 Cache Handle: 0x1003
        Serial Number: A5F9B0AD-E023-7E89-CF01-47772188AD003
        Asset Tag: 9EEC0F35-D6DB-EE11-4788-C0EE56755439
        Part Number: ABD15C29-35D3-1659-BFAF-AD57F39874C3
        Core Count: 64
        Core Enabled: 64
        Thread Count: 64
        Characteristics:
                64-bit capable
                Multi-Core
                Execute Protection
                Enhanced Virtualization
                Power/Performance Control
```
CPU支持过序列号功能，但是被人指责侵犯隐私，所以现在的规范中，CPU完全没有所谓的序列号。

关于CPU序列号，其实还有一段历史。在奔腾3中短暂的引入过这个功能，但是后来很快就移除了。

EAX=3: Processor Serial Number

See also:  Pentium III § Controversy about privacy issues（https://en.wikipedia.org/wiki/Pentium_III#Controversy_about_privacy_issues）

This returns the processor's serial number. The processor serial number was introduced on Intel Pentium III, but due to privacy concerns, this feature is no longer implemented on later models (PSN feature bit is always cleared). Transmeta's Efficeon and Crusoe processors also provide this feature. AMD CPUs however, do not implement this feature in any CPU models.

For Intel Pentium III CPUs, the serial number is returned in EDX:ECX registers. For Transmeta Efficeon CPUs, it is returned in EBX:EAX registers. And for Transmeta Crusoe CPUs, it is returned in EBX register only.

Note that the processor serial number feature must be enabled in the BIOS setting in order to function.

所以，我们不应该使用CPU Serial Number来作为设备唯一性判断，而应该使用CPU ID来判断。

### 1.Windows下获取CPU ID
如果是windows系统，根据MSDN文档：http://msdn.microsoft.com/en-us/library/aa394373(v=vs.85).aspx
ProcessorId

Data type: string

Access type: Read-only

Processor information that describes the processor features. For an x86 class CPU, the field format depends on the processor support of the CPUID instruction. If the instruction is supported, the property contains 2 (two) DWORD formatted values. The first is an offset of 08h-0Bh, which is the EAX value that a CPUID instruction returns with input EAX set to 1. The second is an offset of 0Ch-0Fh, which is the EDX value that the instruction returns. Only the first two bytes of the property are significant and contain the contents of the DX register at CPU reset—all others are set to 0 (zero), and the contents are in DWORD format."

可以用如下代码获取CPU ID
```c
#include "stdafx.h"
#include<iostream>
 
int main()
{
 
    int32_t deBuf[4];
 
    __cpuidex(deBuf, 01, 0);
    printf("%.8x%.8x", deBuf[3], deBuf[0]);
 
    getchar();
    return 0;
}
```
本地没有msvc编译环境，就不做测试了。


### 2.linux x86/amd64获取CPU ID
在Linux上呢，我们也可以用C内联汇编来实现
```c
#include <stdio.h>
static inline void native_cpuid(unsigned int *eax, unsigned int *ebx,
                                unsigned int *ecx, unsigned int *edx)
{
        /* ecx is often an input as well as an output. */
        asm volatile("cpuid"
            : "=a" (*eax),
              "=b" (*ebx),
              "=c" (*ecx),
              "=d" (*edx)
            : "0" (*eax), "2" (*ecx));
}

int main(int argc, char **argv)
{
  unsigned eax, ebx, ecx, edx;

  eax = 1; /* processor info and feature bits */
  native_cpuid(&eax, &ebx, &ecx, &edx);

  printf("stepping %d\n", eax & 0xF);
  printf("model %d\n", (eax >> 4) & 0xF);
  printf("family %d\n", (eax >> 8) & 0xF);
  printf("processor type %d\n", (eax >> 12) & 0x3);
  printf("extended model %d\n", (eax >> 16) & 0xF);
  printf("extended family %d\n", (eax >> 20) & 0xFF);

  /* EDIT */
  eax = 3; /* processor serial number */
  native_cpuid(&eax, &ebx, &ecx, &edx);

  /** see the CPUID Wikipedia article on which models return the serial 
      number in which registers. The example here is for 
      Pentium III */
  printf("cpu serial number 0x%08x%08x\n", edx, ecx);

```
native_cpuid这段代码来自linux kernel里的源码，其实gcc里有cpuid.h这个文件，它封装了ASM代码，直接引入即可。

看下运行结果：
```bash
[root@localhost xx]# gcc cpu_x86.c -o cpu_x86
[root@localhost xx]# ./cpu_x86
stepping 4
model 5
family 6
processor type 0
extended model 5
extended family 0
serial number 0x0000000000000000
```

如上所示，eax, ebx, ecx, edx这四个寄存器对应的内容就是cpu id。跟dmidecode的结果比较下，可以对应上。

```bash
[root@localhost xx]# dmidecode -t 4
# dmidecode 3.0
Getting SMBIOS data from sysfs.
SMBIOS 2.7 present.

Handle 0x0004, DMI type 4, 42 bytes
Processor Information
        Socket Designation: CPU #000
        Type: Central Processor
        Family: Unknown
        Manufacturer: GenuineIntel
        ID: 54 06 05 00 FF FB AB 0F
        Version: Intel(R) Xeon(R) Gold 6152 CPU @ 2.10GHz
        Voltage: 3.3 V
        External Clock: Unknown
        Max Speed: 30000 MHz
        Current Speed: 2100 MHz
        Status: Populated, Enabled
        Upgrade: ZIF Socket
        L1 Cache Handle: 0x0016
        L2 Cache Handle: 0x0018
        L3 Cache Handle: Not Provided
        Serial Number: Not Specified
        Asset Tag: Not Specified
        Part Number: Not Specified
        Core Count: 1
        Core Enabled: 1
        Characteristics:
                64-bit capable
                Execute Protection

```
### 3.aarch64下获取CPU ID

如果是aarch64架构，CPU架构不一样，就不能用同样的ASM汇编了，找了下ARM官方文档，https://developer.arm.com/documentation/ddi0500/d/system-control/aarch64-register-descriptions/main-id-register--el1?lang=en，参考CPU架构，可以从MIDR_EL1寄存器获取
```c
#include <stdio.h>

int main(int argc, char **argv)
{
   unsigned long arm_cpuid;
  __asm__("mrs %0, MIDR_EL1" : "=r"(arm_cpuid));
  printf("%-20s: 0x%016lx\n", "MIDR_EL1=", arm_cpuid);
}
```
输出如下
```
[root@master98 xx]# gcc cpu.c -o cpu
[root@master98 xx]# ./cpu
MIDR_EL1=           : 0x00000000701f6633
```
正好与dmidecode中的ID对应。经过测试，重启后cpuid是不会改变的。

### 4.CPU ID or Serial Number？

Java代码里匹配的是Serial Number，这里一直说的是CPU ID，这俩东西到底是不是同一个事呢？

结论是：
1.CPU Serial Number是一个Embedded 96-bit code during chip fabrication，但废弃标准，不应该使用，而应该使用CPU ID来判断。

2.因为涉及隐私问题（Serial Number is Readable by networks & applications），现在的服务器架构已经不支持CPU Serial Number的获取了，用dmidecode获取到的Serial Number不保证有值的。

3.CPU ID包含的是CPU架构的一些信息，更接近条形码的概念，并不是唯一身份标识，不保证唯一性。

4.dmidecode在国产服务器架构下获取到的CPU Serial Number，其实又叫PSN（Processor Serial Number）。之所以国产化服务器能拿到PSN，是因为国产服务器是aarch64架构，并且是自主化研发，并没有遵循Intel的规范。另外同为国产化服务器，不同的厂家实现也不一样，有的重启即变，有的并不会变化。关于PSN的开启，应该是可以在BIOS里配置。其实，PSN should NOT exist at all。为什么国产服务器还保留PSN，就不做过多展开了。有兴趣的可以自行阅读PSN相关文档


最后，修改很简单，如果使用场景不严格，可以使用CPU ID，或者System Information中的UUID即可，两者都能保证重启不变，但System Information中的UUID能保证唯一性，而CPU ID不能 。