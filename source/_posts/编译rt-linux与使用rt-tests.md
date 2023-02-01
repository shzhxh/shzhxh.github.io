---
layout: post
title:  "编译rt-linux与使用rt-tests"
date:   2021-11-14 22:02:00 +0800
categories: Linux
---

### 1 rt-linux

#### 1.1 下载内核与补丁

```shell
# 内核下载地址：https://mirrors.edge.kernel.org/pub/linux/kernel/
# 补丁下载地址：https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/
# 通过与本地内核版本对比，最接近的应该是5.4.154，故下载此版本的内核与补丁
mkdir rt_kernel && cd rt_kernel
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.154.tar.xz
wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/patch-5.4.154-rt65.patch.xz
tar xf linux-5.4.154.tar.xz
cd linux-5.4.154
xz -dc ../patch-5.4.154-rt65.patch.xz | patch -p1
```

#### 1.2 配置内核

```sh
# 使用make XXXconfig可以生成编译需要的配置文件，这个配置文件是.config。make oldconfig基于已有的.config文件生成新的配置文件，这样就可以避免配置大量的内核参数了。内核版本相差太大，可能出现内核符号不一致的情况，所以上一步需要让内核的版本尽量一致。
cp /boot/config-5.4.0-87-generic linux-5.4.154/.config
sudo apt install libncurses-dev bison flex bc libelf-dev libssl-dev
make oldconfig	# 选Fully Preemptible Kernel (RT)
		# CONFIG_DEBUG_INFO的作用是把调试信息加到内核，这会使内核变大。可以赋值N，以减少内核大小。
	# CONFIG_SYSTEM_TRUSTED_KEYS的作用是指定默认的系统keyring，对于Ubuntu的内核它是'debian/canonical-certs.pem'，然而我们要编译的内核没有这个文件，所以要把它置空，否则会产生编译错误。
```

#### 1.3 编译内核

```sh
# make deb-pkg用于构建deb内核包，即包括源码包也包括二进制包。
make -j4 deb-pkg	# 我的CPU是4核，故选-j4
```

#### 1.4 安装deb包

```
sudo dpkg -i ../*.deb
```

### 2 rt-tests

rt-tests是一个用于测试各种Linux的实时特性的测试套件。

#### 2.1 编译与安装

```
sudo apt-get install build-essential libnuma-dev	# 安装编译环境和必需的库
git clone git://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git
cd rt-tests
git checkout stable/v1.0	# master分支不是稳定版，所以要切换到stable分支
make all
make install
```

#### 2.2 用法

#####  [cyclictest](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)

通过精确测量线程的预期唤醒时间和实际唤醒时间，以提供系统时延的统计信息。这些时延是由硬件、固件和操作系统引起的。

cyclictest通过运行一个非实时的主线程(调度类型SCHED_OTHER)来测试时延，它会启动固定数量的度量线程，这此度量线程有固定的实时优先级(调度类型SCHED_FIFO)。度量线程被在固定间隔内被计时器周期性地唤醒。随后，预期唤醒时间和实际唤醒时间通过共享内存传给主线程。主线程追踪时延的值并打印出它们。

```
cyclictest [options]
# 选项
-a, --affinity[=PROC-SET]	# 指定哪些核来运行线程。不指定[PROC-SET]，则使用所有的核。 
-d, --distance=<DIST>	# 线程间隔之间的距离，单位us，默认值500。当每个核上只有1个线程，建议设置为0.当每个核上有多个线程，用来隔开线程唤醒的时间，这个值会累加到interval的值上。
-h, --histogram=<US>	# 输出时延直方图(latency histogram)的数据，这些数据可以用来生成一个时延直方图。<US>是要跟踪的最大时延，以毫秒为单位。此选项会使所有线程的优先级相等。
-i, --interval=<INTV>	# 线程间的基本间隔，单位微秒，默认值1000。即线程多久被唤醒1次。
-l, --loops=<LPS>	# 设定循环的次数。默认为0(无限循环)。
-m, --mlockall		# 锁定当前及未来的内存分配，以防止被换页的时候换出
-n, --nanosleep		# 使用clock_nanosleep运行测试，这样精度就更高了。如不用此选项，则使用posix间隔计数器。
-p, --priority=<PRIO>	# 设置第一个线程的优先级。
-S, --smp		# 对SMP系统的标准测试选项。相当于"-t -a -n"，且让所有线程的优先级保持相等
-t, --threads[=num]	# 设置测试线程的数量。如果不指定[num]，则为处理器的核心数。若不使用此选项，则线程数为1.
# 示例
sudo cyclictest -t 5 -p 80 -n	# 5个线程，最高优先级80，无限循环
sudo cyclictest -t 1 -p 99 -n -l 1000 -m -i 2 -h 100	# 1个线程，优先级99，循环1000次，输出时延直方图
sudo cyclictest -m -S -p 80 -i 200 -d 0	# 测试smp系统的时延
```

结果的含意：

| 缩写 | 含义     | 描述                               |
| ---- | -------- | ---------------------------------- |
| T    | Thread   | 线程索引和线程ID                   |
| P    | Priority | RT线程的优先级                     |
| I    | Interval | 度量线程的预期唤醒周期，单位us     |
| C    | Count    | 时延被测量的次数，例如：迭代次数   |
| Min  | Minimum  | 测量到的最小时延，单位us           |
| Act  | Actual   | 最近完成的迭代测量到的时延，单位us |
| Max  | Maximum  | 测量到的最大时延，单位us           |

#####  [hackbench](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/hackbench)

hackbench用来定位系统中的瓶颈，这些瓶颈应该被消除或优化。

hackbench测试的是内核里的调度器。它还可以通过重复设置和删除线程来对内存子系统进行压力测试。此外，它还可以在一定程度上对进程间通信（如本地的网络插座，管道等）进行压力测试。

它可以测试系统的负载，但不能精确测试应用的负载，因为它不测试与设备的通信。

它创建若干对可调度实体（进程或线程），每对调度实体之间通过网络插座或管道通信，它会记录每对之间来回发送数据的时间。

```
hackbench [options]
# 选项
-p, --pipe	# 使用管道发送数据。（默认用网络插座发送数据。）
-s, --datasize=<SIZE>	# 每条消息的大小。默认100字节。
-l, --loops=<NUM>	# 每个实体发送消息的数量。默认100条。
-g, --groups=<NUM>	# 指定可调度实体的对数。默认10对。
-f, --fds=<NUM>	# 每个实体要用的文件描述符数量。默认40个。
-T, --threads	# 创建的是线程。
-P, --process	# 创建的是进程。默认参数。
--help
# 示例
hackbench	# 创建10对进程，每个进程都通过网络插座发100条数据，每条数据包含100字节，每个进程使用40个文件描述符。
hackbench -p -T	# 创建10对线程，每个线程都通过管道发100条数据，每条数据包含100字节，每个线程使用40个文件描述符。
hackbench -s 512 -l 200 -g 15 -f 25 -P	# 创建15对进程，每个进程都通过网络插座发200条数据，每条数据包含512字节，每个进程使用25个文件描述符。
```

##### hwlatdetect

本程序用来控制内核模块hwlat_detector.ko，这个内核模块是用来探测硬件时延的，注意这个时延和Linux内核自身无关。

本程序刚开始是用来探测x86上的SMIs（系统管理中断）。SMIs不由内核管理，内核一般也感知不到它。SMIs由BIOS设置和管理，一般用于关键事件，如热传感器和风扇的管理。有时，SMIs也用来管理那些占用时间过长的任务。

hwlat_detector.ko模块的工作原理：通过调用`stop_machine()`占用CPU的所有可配置时间，轮询TSC（时间戳计数器），然后查找TSC之间的空隙。由于机器已经停止了，中断也关闭了，所有的间隙都表示轮询被中断的时间，只有SMI能做到了。

本程序管理着debugsf的挂载、卸载，hwlat_detector.ko模块的加载、卸载。

#####  pip_stress

本程序测试进程间的优先级继承。本程序不接受任何参数，会在运行完毕后直接退出。

本程序会创建三个进程，这三个进程通过共享内存使用一个优先级继承的互斥锁，且都绑定在一个CPU上。最低优先级的进程持有互斥锁，然后中等优先级的进程抢占它，然后最高优先级的进程抢到这个互斥锁。由于优先级继承互斥锁使用的低优先级进程借给高优先级进程的优先级来解锁互斥锁，这就会阻止中优先级进程阻塞高优先级进程。

#####  pi_stress

对互斥锁的优先级继承的压力测试。它作为时间优先级任务运行，并启动inversion machine thread groups。每个inversion group都会导致优先级反转条件，这样如果优先级继承不起作用则会死锁。

注：pi_stress线程是作为SCHED_FIFO或SCHED_RR线程运行的，这意味着它们可以使系统关键线程饥饿。建议在pi_stress之前把系统关键线程的调度策略改为SCHED_FIFO，并使用10以上的优先级，以避免pi_stress使这些线程饥饿。

```
pi_stress [options]
# 选项
-i, --inversions=<n>	# 指定反转条件的数量，即所有inversion group可以反转条件的总数。默认是-1代表无限次。
-t, --duration=<n>		# 指定测试运行时间为n秒。
-g, --groups=<n>		# inversion group的数量，默认10.
-d, --debug		# 调试模式，会输出大量信息
-v, --verbose	# 输出详细信息
-s, --signal	# 在接收到SIGTERM信号(Ctrl+C)后结束。默认按下任意键结束。
-r, --rr		# 让inversion threads作为SCHED_RR(round-robin)线程运行。默认是SCHED_FIFO。
-p, --prompt	# 在真正进行压力测试之前进行提示
-u, --uniprocesor	# 用一个处理器运行所有的线程。默认是inversion threads在一个处理器，admin threads在其它的处理器。
-m, --mlockall	# 锁定当前及未来的内存分配，以防止被换页的时候换出
-h
```

#####  pmqtest

开启线程对，使用POSIX消息队列测量进程间通信的时延。线程对之间通过`mq_send()/mw_receive()`同步，测量的是发送和接收消息的时延。

#####  ptsematest

开启两个线程，使用POSIX互斥锁测量进程间通信的时延。两个线程之间通过`pthread_mutex_unlock()/pthread_mutex_lock()`同步，测量的是释放和获取锁的时延。

#####  rt-migrate-test

实时任务迁移程序。测试任务在多处理器上的实时调度，以确保最高优先级的任务可以运行在所有CPU上。

#####  signaltest

信号往返测试

#####  sigwaittest

启动两个线程，或fork两个进程，测量它们之间收发信号的时延

#####  svsematest

启动两个线程，或fork两个进程，测量信号量SYSV的时延。
