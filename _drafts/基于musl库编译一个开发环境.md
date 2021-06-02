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

