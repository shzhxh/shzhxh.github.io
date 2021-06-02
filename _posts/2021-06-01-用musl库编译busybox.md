---
layout: post
title:  "用musl库编译busybox"
date:   2021-06-01 22:02:00 +0800
categories: musl
---

参考了这位老哥[Tim Tassonis](https://www.openwall.com/lists/musl/2014/08/08/13)的文档，虽然是2014年写的，但并不过时。

编译平台：运行于qemu-system-riscv64上的Debian

过程如下：

```bash
# 安装必要的软件
sudo apt install build-essentials
sudo apt install musl-tools

# gcc是到gcc-10的链接，现在要让它变成musl-gcc的链接
rm /usr/bin/gcc	
ln -s /usr/bin/musl-gcc /usr/bin/gcc
# musl-gcc使用了cc，要让cc链接到正确的编译器gcc-10
rm /usr/bin/cc	
ln -s /usr/bin/gcc-10 /usr/bin/cc

# 需要包含一些linux头文件
ln -s /usr/include/linux /usr/include/riscv-linux-musl/
ln -s /usr/include/asm-generic /usr/include/riscv-linux-musl/asm
ln -s /usr/include/mtd /usr/include/riscv-linux-musl/
cp /usr/include/riscv64-linux-gnu/asm/byteorder.h /usr/include/asm-generic

# 编译busybox
wget https://busybox.net/downloads/busybox-1.33.1.tar.bz2
tar xf busybox-1.33.1.tar.bz2 && cd busy*
vi menuconfig	# 把"CC = gcc"改为"CC = gcc-10"，为了使下一条命令正确执行
make menuconfig	# 默认动态编译，如需静态编译则设置CONFIG_STATIC=y
vi menuconfig	# 把"CC = gcc-10"改为"CC = gcc"，为了基于musl库编译busybox
make
```

