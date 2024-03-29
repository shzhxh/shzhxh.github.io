---
layout: post
title:  "xv6的启动：从进程初始化到多核调度"
date:   2019-12-15 00:00:00 +0800
categories: OS
tags: xv6
---

操作系统处在硬件和用户程序之间，所以它是对硬件的抽象。前一篇分析的printf函数是对uart的抽象，地址映射是对内存的抽象。本篇准备分析启动过程的其它部分，这些也都是对某种硬件的抽象。

<!-- more -->

#### 一、 进程的初始化

procinit函数定义在kernel/proc.c，它的作用是初始化进程表。个人认为进程就是对CPU的抽象，进程表就是这种抽象的表现形式。xv6对各种硬件进行抽象都有相同的套路：首先要有一个自旋锁，其次把相关数据结构都放在一个数组里，其它部分往往是对前述数组里元素的某种补充。对进程表初始化的核心功能就是为每个进程分配栈空间。最后，使用kvminithart函数刷新TLB，使分配给各个PCB的栈生效。

#### 二、中断的初始化

首先用trapinit函数初始化trap对应的锁。然后trapinithart函数向stvep寄存器写入kernelvec的地址，其中kernelvec是中断处理例程的入口。然后plicinit函数来使能硬件的中断，具体来说就是向PLIC的UART0和VIRTIO0中断使能位写入1。最后plicinithart函数来使能当前核接收中断。

#### 三、文件系统的初始化

首先用binit来初始化缓冲区缓存。然后用iinit来初始化inode缓存。然后用fileinit来初始化文件表。最后用virtio_disk_init函数初始化硬盘。

#### 四、准备第一个用户进程

userinit函数在kernel/proc.c，用于准备第一个用户进程。主要是为用户进程分配资源，并将其状态改为RUNNABLE。当内核开始调度的时候第一个用户进程就开始执行了。

#### 五、多核调度

每个核都会运行到scheduler函数，它用于多核调度。每个核干的事都是在所有进程里找状态为RUNNABLE的进程然后进行它。

- 数组cpus[]的元素是cpu的数据结构，mcpu()返回的是当前cpu所对应的那个cpus[]里的元素。
- intr_on()使能当前核的中断。
- acquire和release分别用于锁的请求和释放。
- swtch用于切换上下文，“返回”用户态。