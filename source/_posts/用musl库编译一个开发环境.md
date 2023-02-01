---
layout: post
title:  "用musl库编译一个开发环境"
date:   2021-06-01 22:02:00 +0800
categories: Musl
---

参考了这位老哥[Tim Tassonis](https://www.openwall.com/lists/musl/2014/08/08/13)的文档，虽然是2014年写的，但并不过时。

<!-- more -->

编译平台：运行于qemu-system-riscv64上的Debian

编译环境的准备

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
```

编译busybox

```bash
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

编译musl库的测试代码

```bash
git clone git://repo.or.cz/libc-test && cd libc-test
make	# 编译成动态和静态两种测试程序，并执行
	# 编译无误且运行正确不输出，否则输出到src/REPORT

# 目前在Alpine on Riscv下没有make命令，故需生成运行脚本
echo "#!/bin/busybox sh" > run.sh
make -n | grep "src/common/runtest" >> run.sh
make -n | tail -4 >> run.sh

# 把libc-test复制到Alpine里，执行测试命令
cd libc-test
. ./run.sh
```

编译lua

```bash
wget http://www.lua.org/ftp/lua-5.4.3.tar.gz
tar xf lua-5.4.3.tar.gz && cd lua*
make posix	# 动态编译。由于之前的准备，现在gcc就是musl-gcc
make posix CC="gcc -static"	# 静态编译
```

busybox自带vi

编译lmbench

```bash
# 下载源码，二选一
git clone https://salsa.debian.org/ahs3/lmbench	# 推荐，在debian下编译很整洁，没有提示异常
git clone https://github.com/intel/lmbench.git	# 编译时会提示一些异常，但可用

# 如直接运行，会提示脚本scripts/gnu-os "unable to guess system type"，进而运行失败，故需替换这个脚本
cd lmbench/scripts
wget 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD' -O config.guess
wget 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD' -O config.sub
mv gnu-os gnu-os.bak	# 保留旧脚本
mv config.guess gnu-os	# 用新脚本替换旧脚本
chmod +x gnu-os			# 给新脚本增加可执行权限

# 运行lmbench
cd lmbench
make results	# 编译并执行
make see		# 查看结果
```

