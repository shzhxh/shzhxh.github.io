---
layout: post
title:  "xv6的系统调用"
date:   2019-12-22 00:00:00 +0800
categories: xv6
---

在xv6启动的过程中，0号核在main函数里会执行userinit函数，标记initcode处的数据会复制到第一个用户进程的内存空间。标记initcode的数字是一段代码，它对应的内容在user/initcode.S，是要把文件init的内容放到内存里以用户态去执行。文件init编译自user/init.c。

user/init.c把文件sh放到内存里执行。文件sh编译自user/sh.c，把用户输入的命令调入内存中执行，它就是xv6的shell。

系统调用怎么从用户态到的内核态呢？文件user/usys.pl生成user/usys.S，usys.S就是用来完成从用户态到内核态的代码。从代码可见，它是通过ecall产生一个异常来进入内核态，在kernelvec处保存寄存器的上下文，调用kerneltrap，

#### 进程和内存

##### fork

文件位置：kernel/proc.c。作用：创建一个进程。

