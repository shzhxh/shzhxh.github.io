---
layout: post
title:  "编译zephyr"
date:   2022-01-03 21:38:00 +0800
categories: OS
---

主要分成了5步，如下所示：

<!-- more -->

```bash
# 1. 添加kitware APT仓库
wget https://apt.kitware.com/kitware-archive.sh
sudo bash kitware-archive.sh
# 2. 安装依赖
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget \
  python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file \
  make gcc gcc-multilib g++-multilib libsdl2-dev
# 3. 下载zephyr
pip3 install --user -U west
west init zephyrproject
cd zephyrproject
west update
west zephyr-export
pip3 install --user -r ./zephyr/scripts/requirements.txt
# 4. 安装工具链
cd ~
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.13.2/zephyr-sdk-0.13.2-linux-x86_64-setup.run
chmod +x zephyr-sdk-0.13.2-linux-x86_64-setup.run
./zephyr-sdk-0.13.2-linux-x86_64-setup.run -- -d ~/zephyr-sdk-0.13.2y
sudo cp sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /etc/udev/rules.d
sudo udevadm control --reload
# 5. 
cd ~/zephyrproject/zephyr
west build -b qemu-riscv64 samples/synchronization
west build -t run
```
