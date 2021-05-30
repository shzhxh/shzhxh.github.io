---
layout: post
title:  "从xv6的启动到内存初始化"
date:   2019-12-08 00:00:00 +0800
categories: xv6
---

这里所指的xv6是指它的riscv版，而不是它的x86版。我用的操作系统是Ubuntu 18.04。

#### 一、 运行前的准备

```
git clone https://github.com/mit-pdos/xv6-riscv.git	# 下载xv6源码
sudo apt install gcc-8-riscv64-linux-gnu	# 安装编译工具
# 需自行编译qemu,编译目标为riscv64-softmmu
cd xv6-riscv
vim Makefile	# 把第51行的"CC = ..."里的"gcc"改为"gcc-8"
```

#### 二、运行xv6

```
cd xv6-riscv
make qemu 
```

编译主要是生成kernel和fs.img两个文件。kernel是从kernel/目录下的文件编译出来的，它是xv6的内核。fs.img是一个镜像文件，包含了一些用户态的程序，这些程序是从user/目录编译出来的。这些用户态的程序，通过mkfs\目录下的mkfs.c生成的程序mkfs，放到fs.img里。

#### 三、内核的入口

kernel/kernel.ld是把内核链接起来的配置文件，在这个文件里指明了内核的入口是`_entry`。

```
ENTRY( _entry )
```

`_entry`在文件kernel/entry.S，当前运行在机器态，这段代码的作用是为C代码的运行做准备，一旦做好准备就跳转到C代码的start函数开始执行。具体来看，它就是为每个核准备了4K大小的栈空间，这些栈的空间都在是kernel/start.c函数里分配的。

```
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];
```

#### 四、跳转到C

`start`函数在kernel/start.c，它的作用是为进入S特权级做好准备，一旦准备完成就进入S态，从`main`函数开始执行。

- 设置mstatus寄存器告诉它我们是从S态进入的，设置mepc寄存器告诉它我们在S态的进入点是main，这样在执行mret指令后处理器就会“返回”到S态，从main函数开始执行。
- 设置satp寄存器为0是告诉处理器在进入S态后不开启分页。
- 设置medeleg和mideleg，告诉它们在S态处理异常和中断。所有的处理都放在S态，这样提升了效率，降低了代码的复杂度。
- timerinit函数用来设置计时器中断。
- 把当前核的编号放在tp寄存器里，这是为内核函数cpuid准备的。
- 执行mret指令，跳转到S态。

#### 五、内核的总控函数main

`main`函数在kernel/main.c，处理器从这个函数开始进入了S态，这个函数是内核的总控函数，它完成了硬件及内核相关数据结构的初始化，最后以处理机调度的状态一直存在着。main函数之后，用户态的第一个进程开始执行了。

#### 六、与外界交互

`main`函数首先要做的事是与外界交互，这样才能看到内核运行的状况，方便定位问题。实现这个功能的函数是`consoleinit`, `printfinit`, `printf`。

- consoleinit是从硬件的层面初始化输入输出。首先，它初始化了控制台的锁cons.lock，这是为了避免多个核同时使用控制台引发输入输出的混乱。然后，初始化uart。最后，把consoleread和consolewrite连接到系统调用read和write。
- printfinit函数的作用是初始化printf函数的锁pr.lock，这也是为了避免多个核同时使用printf函数引发输入输出的混乱。
- printf函数依赖于consputc，consputc依赖于uartputc。可见，对uart的初始化是实现printf函数的基础。

#### 七、内存的初始化

内存的初始化就是建立虚拟内存和物理内存之间的映射关系。实现这个功能的函数是`kinit`, `kvminit`,`kvminithart`。

- kinit把空闲内存加到了一个单向链表freelist里，这样就可以通过链表freelist来管理这些空闲内存了。
- kvminit是为各硬件和内核的物理内存创建到虚拟内存的映射。首先，为空闲页表kernel_pagetable分配一个空闲页；然后，为硬件uart、virtio硬盘、clint和plic建立映射；然后，为内核的代码段和数据段建立映射；最后，为管理内核/用户空间切换的代码空间建立映射。
- kvminithart用于开启分页。使用w_satp函数向satp寄存器写入内核页表kernel_pagetable，它采用的是Sv39的分页方式。但这时分页机制并没有真正生效，需要执行sfence_vma函数来刷新TLB使它生效。



