---
layout: post
title:  "OSDI一：编译环境"
date:   2022-02-20 20:38:00 +0800
categories: OS
---

想深入学习操作系统，不喜欢只有理论而没有实践的书，于是选了这本：操作系统的设计与实现(Operating Systems Design and Implementation)。

第一步是把编译环境建立起来。先是从github上下载源码编译内核，发现在ubuntu20.04环境里是不能编译的，它一上来就是要删除`/usr/include`目录。然后在ubuntu20.04的docker里编译，让它随便折腾去吧，又发现它的C代码不满足现代的规范。在准备放弃的时候，才发现正确的打开方式。原来它是要在minix3自已的环境里编译自己的。

1. 在[这个网址](https://wiki.minix3.org/doku.php?id=www:download:previousversions)下载OSDI这本书所对应的minix3：minix-3.1.0-book.iso.bz2，解压后是个iso文件。

2. 按[这个网址](https://wiki.minix3.org/doku.php?id=usersguide:runningonqemu)的描述创建qemu的启动硬盘。

   ```bash
   qemu-img create minix.img 8G	# 创建一个硬盘镜像
   qemu-system-x86_64 -net user -net nic -m 256 -cdrom minix.iso -hda minix.img -boot d	# 在qemu上执行安装过程
   # 输入root登入系统，输入setup执行安装过程
   poweroff	# 安装完毕，退出系统
   off			# 在接下来的提示符输入此命令退出qemu
   ```

3. 启动minix3

   ```bash
   qemu-system-x86_64 -net user -net nic -m 256 -hda minix.img
   # 用户名依然是root
   ls /usr/src		# 书上所需源码在此目录
   which cc aal	# 编译环境已具备
   ```

   

