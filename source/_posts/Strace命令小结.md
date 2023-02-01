---
layout: post
title:  "Strace命令小结"
date:   2021-05-30 22:02:00 +0800
categories: Linux
---

#### 概述

`strace`用于追踪某个程序所使用的系统调用和所接收到的信号。

<!-- more -->

它的用法是：

```bash
strace [options] <command> [args]
```

其中`command`就是所要运行的命令，`args`是`command`的参数。而`options`是`strace`命令的参数。

在显示系统调用的时候，输出的格式为：系统调用名 + 传递的参数 + “=” + 返回值。

在显示信号的时候，输出的格式为：信号的符号+ `siginfo`结构体。

#### 选项

把信息输出到文件：

```bash
-o <filename>	# 把输出写入文件filename，而不是写到stderr.
```

增加输出的信息：

```bash
-d	# 显示一些调试信息。
-i	# 打印出系统调用的指针。
-t	# 打印出系统调用的时间，精确到秒。
-tt	# 打印出系统调用的时间，精确到毫秒。
-ttt	# 打印出epoch以来的秒数+系统调用的时间(精确到毫秒)。
-T	# 打印系统调用所花费的时间。
```

减少输出的信息：

```bash
-e <expr>	# 用expr来定义要追踪的事件及追踪它的方式。
	# expr的格式：[qualifier=][!][?]<value1>[,[?]value2...]
-e trace=<set>	# 指定要追踪的系统调用。默认trace=all。
	# set指系统调用的集合，如：open,close,read,write
-e trace=%file	# 追踪以文件名为参数的系统调用
-e trace=%process	# 追踪进程管理的系统调用
-e trace=%network	# 追踪网络相关的系统调用
-e trace=%signal	# 追踪信号相关的系统调用
-e trace=%ipc		# 追踪IPC相关的系统调用
-f	# 仅追踪子进程。这些子进程就是要追踪的进程，它们一般由fork,vfork或clone系统调用创建。
-v	# 打印出系统调用参数的详细信息。
```

调整输出的格式：

```bash
-c	# 统计每个系统调用的时间、调用次数和错误次数。
-S <sortby>	# 对输出的结果排序。可选参数有time,calls,name,nothing，默认time.
```

