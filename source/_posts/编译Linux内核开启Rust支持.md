---
title: 编译Linux内核开启Rust支持
date: 2023-02-06 14:05:57
categories: Linux
tags:
---

编译Rust for Linux过程记录

<!-- more -->

##### 安装依赖

```shell
git clone https://github.com/Rust-for-Linux/linux.git -b rust --single-branch
cd linux
# 从https://www.rust-lang.org安装rustup
rustup override set $(scripts/min-tool-version.sh rustc)	# rustc
rustup component add rust-src	# rust-src
sudo apt install clang llvm		# libclang
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
rustup component add rustfmt	# rustfmt
rustup component add clippy		# clippy
make LLVM=1 rustavailable		# 验证如上依赖安装无误，输出“Rust is available!”
```

##### 配置与编译

```shell
# for amd64
make LLVM=1 menuconfig
	# Kernel hacking -> Sample kernel code -> Rust samples 选择一些模块
make LLVM=1 -j

# for aarch64
make ARCH=arm64 CLANG_TRIPLE=aarch64_linux_gnu LLVM=1 menuconfig
make ARCH=arm64 CLANG_TRIPLE=aarch64_linux_gnu LLVM=1 -j

# for riscv64
make ARCH=riscv LLVM=1 menuconfig
	# 内核有bug,需关闭"Virtualization -> Kernel-based Virtual Machine (KVM) support"
make ARCH=riscv LLVM=1 LLVM_IAS=0 -j
```

##### 运行

```shell
# 1. 从https://people.debian.org/~gio/dqib/下载预编译好的debian，解压
# 2. 安装内核
#   a. 对于amd64和arm64，把编译出的bzImage(或Image)复制到debian目录
#   b. 对于debian-rv64，把linux目录复制到qemu上的debian，在该目录下执行"make install"安装内核
# 3. 参考readme，从qemu启动内核
#   注：对于debian-rv64,需要安装u-boot-qemu和opensbi
# 4. 运行rust内核模块
#   a. 如果配置的时候选择的是build-in：
      dsesg | grep rust	# 验证rust示例运行起来
#   b. 如果不是build-in而是以模块的形式存在
#     把kernel目录复制到qemu上的debian
       cd linux
       make modules_install	# 安装模块
       modprobe <modname>	# 观察输出
       rmmod <modname>		# 观察输出
```



#### 错误记录

1. 执行`make LLVM=1 rustavailable`出现`./scripts/rust_is_available.sh: 21:  arithmetic expression: expecting primary:`

   原因分析：在`rust_is_available.sh`里加`set -x`可发现是`bindgen_libclang_cversion`为空，因没有安装libclang引起的。

   解决方法：安装clang和llvm

2. menuconfig里没有找到Rust samples

   原因分析：在General setup -> Rust support没有选中，选中它即解决此问题。

3. 执行`make LLVM=1`提示`linker 'ld.lld' not found`

   原因分析：没有安装它，通过`sudo apt install lld`安装

4. 执行`make LLVM=1`提示`'libelf.h' file not found`和`'gelf.h' file not found`

   解决方法：`sudo apt install libelf-dev`

5. 执行`make ARCH=arm64 CROSS_COMPILE=aarch64_linux_gnu LLVM=1 -j`提示`unknown target triple 'aarch64_linux_gnu'`

   解决方法：使用`CLANG_TRIPLE=`

6. 执行`make ARCH=riscv LLVM=1 LLVM_IAS=0 -j`提示`ERROR: modpost: "riscv_cbom_block_size" undefined`

   原因分析：使用gcc也报同样的问题。这是内核bug，待修复。

   解决方法：关闭"Virtualization"以绕过此问题。
