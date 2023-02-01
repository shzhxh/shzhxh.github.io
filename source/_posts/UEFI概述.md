---
layout: post
title:  "UEFI概述"
date:   2022-01-09 21:38:00 +0800
categories: os
---

UEFI的核心是后面的FI（固件接口），即它本身是个协议，需要硬件和软件的支持。

硬件和固件厂商都已提供，对于内核开发者来说，需要做的就是编写一个UEFI应用程序来加载内核。

#### UEFI启动过程

##### 1. SEC(安全验证)

计算机系统加电后进入此阶段。

- 进入固件入口
- 从实模式进入32位平坦模式
- 定位固件中的BFV(Boot Firmware Volume)
- 定位BFV中的SEC映像
- 从32位模式进入64位模式
- 调用SEC入口函数
- 利用CAR(Cache As Ram)技术初始化栈
- 调用PEI入口函数

##### 2. PEI(EFI前期初始化)

将需要传递给DXE的信息组成HOB(Handoff Block)列表，然后将控制权转交到DXE手中。

- 初始化PS(PEI Core Service)
- 调度系统中的PEIM(PEI模块)
- 准备HOB列表
- 调用DXE入口函数

##### 3. DXE(驱动执行环境)

执行大部分系统初始化工作。

- 根据HOB列表初始化系统服务
- 调度系统中的驱动
- 调用BDS入口函数

##### 4. BDS(启动设备选择)

BDS被认为是一种特殊的DXE阶段的应用程序，主要执行启动策略。

- 初始化控制台设备
- 加载必要的设备驱动
- 加载和执行启动项

##### 5. TSL(操作系统加载前期)

OS Loader运行的阶段，OS Loader是一个UEFI应用程序。此时系统资源仍由UEFI内核控制。默认运行OS Loader，但如果用户干预则进入UEFI Shell。当ExitBootServices()服务被调用后，则进入RT阶段。

##### 6. RT(操作系统运行)

OS获得系统控制权。

##### 7. AL(After Life)

为系统固件提供错误处理和灾难恢复机制。
