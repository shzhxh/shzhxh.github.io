---
title: PCIe概述
date: 2023-09-15 09:49:06
categories: Hardware
---

PCI(外围组件互连)和PCIe最大的区别是传输方式：PCI是并行传输，而PCIe是串行传输。因为串行传输可以将传输频率大幅提升，从而提升了传输速度。

<!-- more -->

#### PCI架构

PCI总线主要由三部分组成：

1. PCI设备。分为主设备和目标设备，主设备是访问的发起者，目标设备是被访问者。
2. PCI总线。系统中可以有多条PCI总线。每条PCI总线上可以连接多个PCI设备或PCI桥。
3. PCI桥。PCI桥连接了两条PCI总线。

PCI标准的特点：

1. 它是并行总线。一个时钟周期内传32个bit，地址信息32位，数据信息32位，所以传地址和信息需要2个时钟周期。
2. 处理器空间和PCI空间通过host桥隔离。host桥的存在，使处理器和PCI工作在各自的时钟频率，方便地共享内存资源。
3. 扩展性强。可以通过PCI总线桥扩展出一系列PCI总线，它们形成以root桥为根结点的一棵PCI总线树。

#### PCIe架构

PCIe总线的组成：

1. root port
2. endpoint
3. switch。可以同时连接多个endpoint。
4. PCI总线。一个root port和一个endpoint之间就需要一个单独的PCI总线。
5. lane。PCIe的连线。lane组合起来可以提供更高的带宽，最大直到x16。

PCIe总线的特点：

1. PCIe是点对点结构。
2. 配置空间为4K。(作为对比，PCI的配置空间只有256B)。
3. 提供了很多特殊功能。如CTO(Complete Timeout)，MaxPayload，电源管理State(L0/L0s/L1)等。

#### 配置空间

PCI总线上的设备完全是由软件驱动来初始化和配置的，这是通过各设备的配置空间(Configuration Address Space)来实现的。

PCI设备必须提供配置空间，其大小最高可以有256比特。配置空间的前64个bit(0x00~0x3f)是PCI设备必须支持。0x40~0xff是扩展的配置空间。

##### 通用的header字段

配置空间里通用的header字段。

- device id：由供应商提供设备id。

- vendor id：由PCI-SIG来分配供应商id。

- status：一个寄存器，用于记录PCI总线相关事件的状态信息。

- command：

- class code：一个只读寄存器，定义了设备可以实现的功能的类型。

- subclass：

- prog IF：一个只读寄存器，定义了寄存器级的编程接口。

- revision id：设备的修订标识符。由供应商提供。

- BIST：built-in self test

- header type：配置空间从0x10开始的非通用部分的类型。

  | head type | 含义             |
  | --------- | ---------------- |
  | 0         | 普通设备         |
  | 1         | PCI-to-PCI桥     |
  | 2         | PCI-to-CardBus桥 |
  | 0x80      | 此设备有多个功能 |

- latency timer：在PCI总线的时钟集里指定latency timer。

- cache line size：

##### type0设备的配置空间

type0设备(非bridge设备)的配置空间：

| 寄存器(32bit) | offset(字节) | 名字 (bit: bit)                                              |
| ------------- | ------------ | ------------------------------------------------------------ |
| 0x0           | 0x0          | Vendor id + device id (16 : 16)                              |
| 0x1           | 0x4          | command + status (16 : 16)                                   |
| 0x2           | 0x8          | revision id + prog IF + subclass + class code (8: 8: 8: 8)   |
| 0x3           | 0xc          | cache line size + latency timer + header type + BIST (8 : 8 : 8: 8) |
| 0x4           | 0x10         | BAR0 (base address register #0)                              |
| 0x5           | 0x14         | BAR1                                                         |
| 0x6           | 0x18         | BAR2                                                         |
| 0x7           | 0x1c         | BAR3                                                         |
| 0x8           | 0x20         | BAR4                                                         |
| 0x9           | 0x24         | BAR5                                                         |
| 0xa           | 0x28         | Cardubs CIS Pointer                                          |
| 0xb           | 0x2c         | subsystem vendor id + subsystem id (16: 16)                  |
| 0xc           | 0x30         | XROMBAR(expansion ROM base address register)                 |
| 0xd           | 0x34         | Capabilities pointer + 保留 (8: 24)                          |
| 0xe           | 0x38         | 保留                                                         |
| 0xf           | 0x3c         | Interrupt line + interrupt pin + min grant + max latency (8: 8: 8: 8) |

##### 访问配置空间

PCI设备的编码简称为BDF，即：Bus号 + Device号 + Function号。

访问PCI设备的寄存器(只能访问256比特)：

- CFGADR(配置地址寄存器)，I/O空间地址0xCF8，用于指定要访问的配置地址。

  | 位    | 名字         | 作用                                             |
  | ----- | ------------ | ------------------------------------------------ |
  | 31    | 配置使能     |                                                  |
  | 24~30 | 保留         |                                                  |
  | 16~23 | Bus号        | 选择哪个总线                                     |
  | 11~15 | Device号     | 选择总线上的哪个设备                             |
  | 8~10  | Function号   | 选择设备的哪个功能（如果设备有多个功能）         |
  | 2~7   | 寄存器编号   | 指定配置空间上的寄存器（最多只可能有64个寄存器） |
  | 0～1  | 保留，恒为00 |                                                  |

- CFGDAT(配置数据寄存器)，I/O空间地址0xCFC，用于读取或写入要传输的配置数据。

PCIe的配置空间有4KB，使用I/O空间地址只能访问前255B，使用MMIO可以访问全部的配置空间，其地址记录在ACPI 'MCFG'表。

#### BAR空间

BAR(base address register)，记录了设备所需要的内存地址。

内存空间里BAR的布局：

| bit 31-4                  | bit 3        | bit 2-1 | bit 0 |
| ------------------------- | ------------ | ------- | ----- |
| base address (16字节对齐) | prefetchable | type    | 恒为0 |

I/O空间上BAR的布局：

| bit 31-2                 | bit 1 | bit 0 |
| ------------------------ | ----- | ----- |
| base address (4字节对齐) | 保留  | 恒为1 |

#### PCI桥设备的配置空间

PCI-to-PCI桥设备，即type 1设备，配置空间如下：

| 寄存器(32bit) | offset(字节) | 名字 (bit: bit)                                              |
| ------------- | ------------ | ------------------------------------------------------------ |
| 0x0           | 0x0          | Vendor id + device id (16 : 16)                              |
| 0x1           | 0x4          | connand + status (16 : 16)                                   |
| 0x2           | 0x8          | revision id + prog IF + subclass + class code (8: 8: 8: 8)   |
| 0x3           | 0xc          | cache line size + latency timer + header type + BIST (8 : 8 : 8: 8) |
| 0x4           | 0x10         | BAR0 (base address register #0)                              |
| 0x5           | 0x14         | BAR1                                                         |
| 0x6           | 0x18         | 主总线号+二级总线号+ subordinate总线号+ secondary latency timer (8: 8: 8: 8) |
| 0x7           | 0x1c         | I/O base + I/O limit + secondary status (8: 8: 16)           |
| 0x8           | 0x20         | memory base + memory limit (16:16)                           |
| 0x9           | 0x24         | prefetchable memory base + prefetchable memory limit (16: 16) |
| 0xa           | 0x28         | prefetchable base upper 32 bits                              |
| 0xb           | 0x2c         | prefetchable limit upper 32 bits                             |
| 0xc           | 0x30         | I/O base upper 16 bits + I/O limit upper 16 bits (16: 16)    |
| 0xd           | 0x34         | capability pointer + 保留 (8: 24)                            |
| 0xe           | 0x38         | XROMBAR(expansion ROM base address register)                 |
| 0xf           | 0x3c         | Interrupt line + interrupt pin + bridge control (8: 8: 16)   |

##### capabilities结构

capabilities结构代表了PCIe设备的各种特性，如Max Payload, CTO等。capabilites的结构是：多个capability组成链表结构。

#### 参考资料

- [深入PCI与PCIe之一：硬件篇](https://zhuanlan.zhihu.com/p/26172972)
- [深入PCI与PCIe之二：软件篇](https://zhuanlan.zhihu.com/p/26244141)
- [osdev.org/PCI](https://wiki.osdev.org/PCI)
