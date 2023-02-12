---
layout: post
title:  "Aero操作系统概览"
date:   2022-12-05 20:38:00 +0800
categories: OS
---

 运行Aero操作系统，及其启动过程分析。

<!-- more -->

#### 一 运行Aero

```bash
git clone https://github.com/Andy-Python-Programmer/aero.git
pip3 install requests xbstrap	# 安装aero.py所需要的包
./tools/deps.sh		# 安装运行所依赖的程序，for Arch
sudo apt install xorriso subversion mercurial nasm mtools groff expat help2man xcb-proto libboots-all-dev libfreetype-dev
	# 安装运行所依赖的程序, for debian 11
cp /usr/sbin/zic ~/.local/bin/	# for debian 11
./aero.py --sysroot		# 构建完整的sysroot
./aero.py	# 构建内核和userland,并在qemu里运行它
```

#### 二 启动过程

从`src/.cargo/kernel.ld`可见，`arch_aero_main()`是入口。

对x86_64架构，`arch_aero_main()`在`src/aero_kernel/src/arch/x86_64/mod.rs`。本函数的作用是完成与架构相关的初始化。本函数执行完成后会调用`aero_main()`。

`aero_main()`在`src/aero_kernel/src/main.rs`。本函数仅初始化必需的服务，其它的初始化工作由`kernel_main_thread()`来完成。调度进程会调度`kernel_main_thread()`执行。

1. `fs::init().unwrap()`用于加载文件系统。
2. 加载计时器。
3. 加载调度进程。

`kernel_main_thread()`在`src/aero_kernel/src/main.rs`。

1. 加载内核模块。
2. 加载PCI驱动程序。
3. 执行`fs::block::launch().unwrap();`
4. 执行`userland::run().unwrap();`
