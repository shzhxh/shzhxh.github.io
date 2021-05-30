---
layout: post
title:  "XV6的自陷管理"
date:   2020-04-19 00:00:00 +0800
categories: xv6
---

#### 简介

用自己的语言描述XV6的自陷管理系统。XV6把CPU跳出当前指令流转而执行其它代码的情况叫trap，看到有人把它翻译成**自陷**，感觉这么翻译还是比较准确的。自陷可分为三种情况：

- 系统调用，使用`ecall`指令跳出当前代码流。
- 异常，由于指令非法执行跳出当前代码流。
- 设备中断，由于设备发出中断信号跳出当前代码流。



#### 引子

当xv6在启动的过程中，首先在机器模式下调用`timerinit`初始化了计时器中断，然后切换到管理员模式下，主控函数`main`调用了两个函数来完成自陷管理系统的初始化。

- `trapinit`打个酱油。
- `trapinithart`指定内核模式下的自陷处理例程。

与自陷管理相关的文件有`kernel`目录下`start.c`、`trap.c`、`kernelvec.S`、`trampoline.S`。

- `start.c`处理计时器中断(机器模式下)。
- `trap.c`处理管理员模式和用户模式下的自陷。
- `kernelvec.S`处理管理员模式下的自陷和机器模式下的计时器中断。
- `trampoline.S`用户模式到管理员模式，以及管理员模式返回用户的跳板。

#### 管理员模式下的自陷

管理员模式下只可能发生两种自陷：异常和设备中断。

当自陷发生，硬件会把`stvec`的值保存到`pc`，进而执行`stvec`所指向的地址。`stvec`里保存的是`kernelvec`的地址，所以自陷发生后执行`kernelvec`。`kernelvec`做的事是保存和恢复上下文，其它的工作交给C代码的`kerneltrap`来处理。`kerneltrap`做的事是排查异常情况，自陷处理工作交给`devinitr`来做。`scause`最高位为1说明是来自于中断，此时会判断中断的类型分别处理；否则，最高位就是0，说明来自于异常，此时不做任何处理直接返回0。

#### 用户模式下的自陷

用户模式下三种自陷都有可能发生。

当自陷发生，硬件从用户模式进入到管理员模式，此时`stvec`里保存的是`uservec`的地址(**不知道什么时候把uservec写入到stvec的？**)。`uservec`做的事是保存上下文，切换到内核栈和内核的页表，最后跳转到`usertrap`。`usertrap`做的事是依据自陷的类型，分发给不同的函数去处理，如果是系统调用则执行`syscall`，如果是外设中断则执行`devinitr`，如果是除系统调用外的其它异常则执行`exit`，如果没有执行`exit`则会调用`usertrapret`。`usertrapret`为下次用户空间的自陷做准备，最后切换到`userret`继续执行。`userret`切换到用户页表，恢复上下文，最后通过`sret`指令返回到用户模式。

#### 计时器中断

运行于机器模式下的`timerinit`初始化了计时器中断。由于virtio机器的时钟频率是10MHz，所以`interval`设置为1M的意思就是每100ms产生一个计时器中断。`scratch[]`区域用来给`timervec`提供保存和恢复上下文的空间。`timervec`被保存在`mtvec`里，它的工作是处理计时器中断。`timervec`产生了一个管理员模式下的软件中断，这样就可以通过管理员模式下的自陷处理机制来管理计时器中断了。

#### 参考文档

- [xv6-riscv-book Chapter 4](https://github.com/mit-pdos/xv6-riscv-book)
- riscv-privileged-v1.10