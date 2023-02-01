---
layout: post
title:  "管窥C标准库"
date:   2021-11-28 18:38:00 +0800
categories: Language
---

我觉得C语言是一个马甲，它的里面是体系结构，C标准库也是一个马甲，它的里面是操作系统内核。

<!-- more -->

Linux上glibc标准库libc6-dev包含457个头文件，而我只以自己记录过的31个头文件为基准来描述它，只能覆盖头文件的6.7%，所以只能说是管窥了。

- arpa：早期的网络就叫arpa网，是为了向arpa致敬吗？
  - inet.h：ip地址形式之间的转化。
- x86_64-linux-gnu/sys：关于系统调用
  - ipc.h：定义了System V IPC的函数ftok。
  - mman.h：内存映射管理
  - sem.h：SYS V信号量。
  - socket.h：网络插座
  - stat.h：设置文件的状态
  - time.h：设置时间
  - wait.h：wait相关的系统调用
- assert：为程序增加诊断功能
- crypt.h：数据加密
- ctype.h：一些测试字符类别的函数
- fcntl.h：文件操控
- fnmatch.h：文件名匹配
- getopt.h：获取函数参数
- iconv.h：字符集转换
- libgen.h：分解目录名
- locale.h：语言环境管理
- math.h：声明了一些数学函数和宏
- poll.h：轮询等待文件描述符上的事件
- pthread.h：线程管理
- sched.h：管理调度器
- search.h：查找
- semaphore.h：管理信号量
- setjmp.h：跳转
- signal.h：处理信号
- stdio.h：管理输入输出
- stdlib.h：实用函数
- string.h：管理字符串
- strings.h：高级的字符串管理
- termios.h：通用终端的接口
- time.h：日期与时间处理
- unistd.h：Unix标准，包含了符号常量的定义与大量系统调用的封装。
- wchar.h：对宽字符的操作
