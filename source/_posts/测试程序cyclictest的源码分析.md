---
layout: post
title:  "测试程序cyclictest的源码分析"
date:   2021-12-12 21:38:00 +0800
categories: Command
---


#### 背景知识

##### numa

Non-Uniform Memory Access，非一致性内存访问。

<!-- more -->

numa出现之前，所有CPU共用一根总线访问内存(UMA，统一内存访问)。UMA的问题是随着CPU核数增加，访存效率下降。

在numa架构里，有结点的概念。一个结点里包含了若干CPU和内存。结点内部访存快，结点外部访存慢，访存存在着本地和远程的区别。可以使用`numactl --hardware`命令查看系统中的numa结点。

##### smi

System Management Interrupt，系统管理中断。

这是硬件系统的中断，内核是看不到它的。

外部设备通过SMI引脚来触发smi，软件也可以通过端口0xB2触发smi。

#### 代码导读

##### 选项相关变量

是用户选项与源码之间的接口。大致相当于`process_options()`函数的解读。

```c
/* -a选项, --affinity选项 */
affinity_mask;		// 如-a有参数，则通过parse_cpumask()设置
setaffinity;	// 如-a有参数则AFFINITY_SPECIFIED; 如-a无参数则AFFINITY_USEALL；默认值为AFFINITY_UNSPECIFIED。
	// AFFINITY_UNSPECIFIED - 用户没有设置亲和性，即用户不使用亲和性
	// AFFINITY_SPECIFIED - 用户指定了亲和性所对应的cpu核
	// AFFINITY_USEALL - 用户要使用亲和性，但没指定对应的cpu核，此时程序会给线程分配一个cpu核

struct bitmask {
    unsigned long size; /* 掩码应包含多少位，其值等于max_cpus */
    unsigned long *maskp;	/* long整型的数组，掩码在整块数组的内存上依次排列 */
} *affinity_mask;

/* -A选项, --aligned选项。它与--secaligned选项是互斥的。 */
aligned=1;		// 把线程的唤醒时间对齐到一个指定的offset。0无需对齐，1需要对齐。如不使用-A选项则默认为0。
offset;			// 如-A有参数，则为(参数*1000)；如-A无参数则为0。(因为参数的单位是微秒，而offset的单位是纳秒，所以要参数*1000)

/* -b选项，--breaktrace选项 */
tracelimit;		// 意为追踪的限制条件。设置为参数的值，当时延大于设置的值时，则会发送break trace command，单位：微秒。默认值为0。
/* -B选项，--preemptirqs选项 */
tracetype;
/* -c选项，--clock */
clocksel;	// 用于选择时钟。默认值为0。0:CLOCK_MONOTONIC,1:CLOCK_REALTIME。
/* -C选项, --context */
tracetype;
/* -d选项, --distance */
distance;	// 线程interval之间的距离。单位微秒，默认值500(DEFAULT_DISTANCE)。
/* -D选项, --duration */
duration;	// 指定测试程序运行的时间，此变量的单位为秒。(用户输入的参数必须带单位，单位可以是m,h,d)。通过parse_time_string()函数解析用户输入的参数。此全局变量默认值为0。
/* -E选项, --event */
enable_events=1;
/* -f选项, --ftrace */
tracetype=FUNCTION;
ftrace=1;

/* -F选项, --fifo */
use_fifo=1;	// 创建一个有名管道，把统计信息写入到它。0，不创建；1，创建。
fifopath;	// 有名管道的路径名，即用户输入的参数。

/* -H选项, --histofall */
histofall=1;
/* -h选项, --histogram */
histogram;	// 代表了时延直方图里的最大时延。直接设为用户的参数，单位微秒。默认值为0，代表程序不会收集并输出时延的数据。如用户设置了此参数，则程序会收集时延数据，并在运行完成后把它输出到stdout。
/* --histfile */
use_histfile=1;
histfile;

/* -i选项, --interval */
interval;	// 线程进行计时循环的基本间隔。设置为参数的值，单位：微秒。默认值：DEFAULT_INTERVAL(1000)。
/* -I选项, --irqsoff */
tracetype;
tracer;

/* -l选项, --loops */
max_cycles;		// 用户设置的循环次数
/* -m选项, --mlockall */
lockall;
/* -M选项, --refresh_on_max */
refresh_on_max=1;	// 延迟更新屏幕，直到碰到一个新的最大时延。对于低带宽很有用。
/* -n选项, --nanosleep */
use_nanosleep=MODE_CLOCK_NANOSLEEP;
/* -N选项, --nsecs */
use_nsecs;
/* -o选项, --oscope */
oscope_reduction;
/* -O选项，--traceopt, 使用traceopt()函数 */
traceopt_size;
traceptr;

/* -p选项, --priority */
priority;
policy;

/* -P选项, --preemptoff */
tracetype;
tracer;

/* -q选项, --quiet */
quiet=1;
/* -r选项，--relative */
timermode=TIMER_RELTIME;	// 设置计时器的模式为相对模式，即计时器在一个时间间隔到期；默认的模式是TIMER_ABSTIME，绝对模式，即计时器在一个绝对的时间到期。
/* -R选项, --resolution */
check_clock_resolution=1;
/* --secaligned选项。它与-A选项是互斥的。 */
secaligned=1;	// 把线程的唤醒时间对齐到一个完整的秒，加上可选的offset
offset;		// 参数*1000。(因为参数的单位是微秒，而offset的单位是纳秒，所以要参数*1000)

/* -s选项, --system */
use_system = MODE_SYS_OFFSET;
/* -S选项, --smp */
smp = 1;	// 
num_threads = max_cpus;
setaffinity = AFFINITY_USEALL;
use_nanosleep = MODE_CLOCK_NANOSLEEP;

/* -t选项, --threads */
num_threads;	// 要创建的线程数。如用户没有指定，则为CPU的核数。如没有使用-t选项，则线程数为1。
/* --spike */
trigger;	// 峰值触发器的值，单位为微秒。当峰值大于触发器的值，则将相关参数记录到一个链表里。默认值为0，意为不进行记录。
/* 链表里的元素就是这个结构体 */
struct thread_trigger {
        int cpu;
        int tnum;       /* thread number */
        int64_t  ts;    /* time-stamp */
        int diff;		/* 峰值本身 */
        struct thread_trigger *next;
};

/* --spike-nodes */
trigger_list_size;	// 峰值(spike)是保存在一个链表里的，这个全局变量保存了链表的元素个数。只有定义了trigger这个变量才有意义。
/* -T选项, --tracer */
tracetype = CUSTOM;
tracer;

/* -u选项, --unbuffered */
setvbuf();	// 让stdout不关联到缓冲区

/* -U选项, --numa */
// 注：需在构建的时候使用numa选项，此选项才有用。
numa = 1;	// 0表示不支持numa；1表示支持numa。默认为0。

/* -v选项, --verbose */
verbose = 1;
/* -w选项, --wakeup */
tracetype = WAKEUP;
/* -W选项, --wakeuprt */
tracetype = WAKEUPRT;
/* ?选项, --help */
display_help(0);

/* --priospread */
priospread = 1;
/* --latency */
latency_target_value;
/* --notrace */
notrace = 1;
/* --policy，使用handlepolicy()函数 */
policy；
/* --dbg_cyclictest */
ct_debug = 1;
/* --laptop */
laptop = 1;
/* --smi */
smi = 1;	// 需设置ARCH_HAS_SMI_COUNTER，且使用--smi选项，才为1，表示开启smi计数；否则为0，表示不开启smi计数。
/* --tracemark */
notrace = 1; /* using --tracemark implies --notrace */
trace_marker = 1;

/*
 * 其它与选项相关的全局变量
 */
static pthread_barrier_t align_barr;
	// 如设置了aligned或secaligned，则调用pthread_barrier_init()初始化此屏障。
tatic pthread_barrier_t globalt_barr;
	// 如设置了aligned或secaligned，则调用pthread_barrier_init()初始化此屏障。意为全局变量globalt的屏障。globalt的意思是全局的时间。
```

##### 关键数据结构

```c
/* Struct to transfer parameters to the thread */
struct thread_param {
	int prio;	// 线程的优先级
	int policy;	// 线程的调度算法
	int mode;	// mode=use_nanosleep+use_system;
    		// MODE_CYCLIC : 0, 使用进程自己的计时器，通过信号等待。0+0
    		// MODE_SYS_ITIMER : 2, 使用系统的计时器，通过信号等待。0+2
    		// MODE_CLOCK_NANOSLEEP : 1,使用进程的计时器，通过睡眠等待。1+0
    		// MODE_SYS_NANOSLEEP : 3,使用系统的计时器，通过睡眠等待。1+2
    		// MODE_SYS_OFFSET : 2,表示这个位移是关于系统时钟的。仅用于计算，不作为mode可取的值。
	int timermode;	// 计时器的模式。
    		// TIMER_ABSTIME : 计时器在一个绝对时间到期
    		// TIMER_RELTIME : 0, 计时器在一个时间间隔到期
	int signal;	// 当计时器超时的时候给当前线程发的信号。主线程把它设置为SIGALRM
	int clock;	// 当前线程所用的时钟类型。
    		// CLOCK_MONOTONIC - 记录单调时间的时钟，即系统启动到现在的秒数。
    		// CLOCK_REALTIME - 系统范围的时钟，用于测量真实时间。
	unsigned long max_cycles;	// 用户设置的循环次数，等于全局变量max_cycles的值。
	struct thread_stat *stats;
	int bufmsk;	// 缓冲区stat->values和stat->smis的掩码，即二进制全为1的数。
	unsigned long interval;
	int cpu;	// 与亲和性关联的cpu编号。-1，无亲和性。其它整数，线程与此cpu有亲和性。
	int node;	// numa结点。-1表示不使用numa结点。
	int tnum;	// 线程的编号。创建线程用的for循环，即for循环的次数，从0开始。
	int msr_fd;	// msi相关
};

/* Struct for statistics */
struct thread_stat {
	unsigned long cycles;	// 记录线程执行了多少次计时循环
	unsigned long cyclesread;
	long min;		// 线程等待的最小值。单位：微秒。
	long max;		// 线程等待的最大值。单位：微秒。
	long act;		// 线程等待的实际值。单位：微秒。
	double avg;		// 线程等待的平均值。单位：微秒。
	long *values;	// 一个缓冲区，用于记录当前线程所有diff的值。
	long *smis;		// 一个缓冲区，用于记录当前线程所有diff_smi的值。
	long *hist_array;	// 代表时延直方图的数组。数组的元素数即为用户设置的全局变量histogram的值，每个元素都是long类型。
	long *outliers;	// 一个数组，记录了时延直方图溢出时stat->cycles的值
	pthread_t thread;
	int threadstarted;	// 线程是否启动的标志。-1已关闭，>0已启动。
	int tid;		// 内核里的线程id
	long reduce;
	long redmax;
	long cycleofmax;
	long hist_overflow;	// 溢出时延直方图的时延的数量
	long num_outliers;	// 数组stat->outliers的下标，它的意思里这个数组里记录了多少个元素。
	unsigned long smi_count;	// 系统管理中断的次数
};
```



##### main函数流程

1. `process_options()`处理参数。

2. `check_privs()`检查当前进程调度算法是否是实时的，或可切换为实时的。

3. 如设置了`trigger`，则调用`trigger_init()`初始化单向链表`head`。

   1. 此链表的元素为`struct thread_trigger`。

      ```c
      /* Info to store when the diff is greater than the trigger */
      struct thread_trigger {
      	int cpu;
      	int tnum;	/* 线程的编号 */
      	int64_t  ts;	/* time-stamp */
      	int diff;
      	struct thread_trigger *next;	// 下一个元素的指针
      };
      ```

   2. 此链表第一个元素的指针为head，最后一个元素的指针为tail。

   3. current用来指向链表上需要被更新的元素。

   4. 此链表增加元素采用后插法。

4. 如设置了`lockall`，则调用`mlockall(MCL_CURRENT|MCL_FUTURE)`。

5. 调用`set_latency_target()`设置电源管理系统，这是为了降低时延。

   1. `/dev/cpu_dma_latency`要存在。
   2. 把0写入文件`/dev/cpu_dma_latency`。效果为告诉电源管理系统不要切换到高cstate。当文件关闭后，电源管理器的行为将切换回系统默认的状态。目的是为了阻止CPU进入低功耗状态。详见`Documentation/power/pm_qos_interface.txt`。

6. 调用`check_kernel()`检查内核信息。

   1. 使用`uname()`获取内核信息，保存在`struct utsname kname`。
   2. 把`kname.release`的信息分别保存在`maj`, `min`, `sub`里。
   3. 使用上步三个变量控制`kv`、`functiontracer`、`traceroptions`的值。
   4. 返回`kv`的值。
   
7. 调用`setup_tracer()`来设置tracer。

   1. 执行`debugfs_prepare()`设置fileprefix，即debugfs的目录名。
   2. 由于现代内核都大于2.6.28，故只看if分支的代码。
   3. 如果debugfs里tracing_enabled文件存在且tracing_on文件不存在，则向tracing_enabled文件里写入1。
   4. 如果设置了tracetype，则向ftrace_enabled写入1，否则写入0。
   5. 使用`settracer("nop")`把"nop"写入current_tracer文件。
   6. 在一个switch分支里依据tracetype的值，把相应的tracer写入current_tracer文件。
   7. 如设置了enable_events，则调用`event_enable_all()`把1写入debugfs里的events/enable文件。
   8. 向traceroptions文件里写入一些tracer。
   9. 如设置了traceopt_count，则向traceroptions文件写入traceptr数组里的指针元素。
   10. 向tracing_max_latency文件里写入0。
   11. 如latency_hist目录存在，则向latency_hist/wakeup/reset文件写入1。
   12. 如trace_fd==-1，则给trace_fd赋值，如tracing_on存在它对应的文件就是tracing_on，否则它对应的文件是tracing_enable。
   13. 调用`opentracemark_fd()`给tracemark_fd赋值。
   14. 跳过else分支不进行分析，因为现代内核一定大于2.6.28。
   15. 调用`tracing(1)`向trace_fd里写入1。

8. 调用`enable_trace_mark()`打开debugfs下的文件trace_marker。

   1. 执行`debugfs_prepare()`挂载debugfs并把它的路径名赋值给fileprefix。
   2. 执行`open_tracemark_fd()`打开debugfs下的文件trace_marker。

9. 执行`check_timer()`检查时钟的精度。

10. 如果设置了check_clock_resolution则开始检查时钟的精度。目的是看报告时钟精度不能高于测量时钟精度。

    1. 获取时钟精度，将reported_resolution设置为ns表示的精度。
    2. 计算1ms可以调用clock_gettime()多少次，将这个结果保存在times里，如少于1000次按1000次算，如大于100000次按100000次算。
    3. time是一个以`struct timespec`为元素的数组，数组大小为times，调用clock_gettime()在一个for循环里把time数组填满。
    4. 在一个for循环里，用min_non_zero_diff记录下数组time里相邻元素之间的最小差值(这个最小差值不能为0)。这个值即为测量到的时钟精度。
    5. 测量时钟精度不能高于报告时钟精度，否则需输出warn信息。

11. 设置mode，如选项里有-n则use_nanosleep=0B01，如选项里有-s则use_system=0B10。

12. 把信号SIGALRM加到信号屏蔽字上。

13. 调用signal()让sighand()处理信号SIGINT, SIGTERM, SIGUSR1。

    1. 如信号是SIGUSR1，则向stderr输出错误信号，然后返回。
    2. 如是其它两个信号，则shutdown置1。
    3. 如定义了refresh_on_max，则唤醒一个等待在PTHREAD_COND_INITIALIZER上的线程。
    4. 如定义了tracelimit，则向trace_fd文件写入"0"。

14. 为两个关键的数组分配内存。数组parameters包含了每个线程的参数，数组statistics包含了每个线程的统计信息。

15. 在一个for循环里创建子线程：

    1. 使用pthread_attr_init()初始化attr。
    2. 依据setaffinity的值设置cpu的值。
    3. 一般不会编译numa参数，忽略if分支。
    4. 为parameters[i]分配内存并将内存初始化为0。
    5. 为statistics[i]分配内存并将内存初始化为0。
    6. 如设置了histogram，则为statistics[i]里关于直方图的字段分配内存并初始化为0。
    7. 如设置了verbose，则为`statistics[i]->value`分配内存，并设置好它的掩码`par->bufmsk`。
    8. 初始化parameters[i]和statistics[i]的相关字段。一般来说，都是根据用户的输入直接赋值。但要注意的是，如果没有设置histogram，每个线程par->interval的值会相差distance。
    9. 使用pthread_create()创建线程，线程代码为timerthread，线程参数为parameters[i]。

16. 如定义了use_fifo，则创建子线程，子线程的代码为fifothread。

17. 如没有设置shutdown，则进入无限循环中：

    1. 执行policyname()，读取整型的policy，返回对应的字符串到policystr。
    2. 如设置了force_sched_other，则slash="/"、policystr2="other"，否则slash和policystr2都是空串
    3. 如果verbose和quiet都没设置，则打印policy和loadavg的值。
    4. 在一个for循环里打印每个线程的状态，如线程的循环次数达到了用户设定的次数，则allstopped++
    5. 调用usleep()睡眠10000ms，即10s。
    6. 如设置了shutdown或allstopped，则退出此循环。
    7. 如设置了refresh_on_max，则执行pthread_cond_wait()睡眠，等待子线程通过`pthread_cond_signal()`唤醒自己。
    
18. 设置ret为EXIT_SUCCESS。

19. outall标志。除了顺序执行可到此处外，在创建线程的for循环里，如果设置了verbose，则会给`statistic[i]->values`和`statistic[i]->smis`分配内存，如果内存分配失败也会跳转到outall标志。

    1. 将shutdown置位。
    2. 调用usleep()睡眠50000ms，即50s。
    3. 如设置了quiet，则将quiet置2。
    4. 在一个for循环里确保所有的线程都已退出。
    5. 如设置了trigger，则调用trigger_print()打印出结构体`thread_trigger`存储的信息。
    6. 如设置了histogram，则打印相关数据并释放`statistics[i]->hist_arry`和`statistics[i]->outliers`指向的内存。
    7. 如果设置了tracelimit。则打印出所有线程的tid。
    8. 在一个for循环里释放所有`statistics[i]`占用的空间。

20. outpar标志。除了顺序执行可到此处外，在主线程里如果给statistics分配内存的时候失败，也会跳转到outpar标志。本标志的代码是用for循环释放所有的`parameters[i]`的内存。

21. out标志。除了顺序执行可到此处外，如果设置了lockall但`mlockall()`执行失败的时候，或如果给`parameters`分配内存失败的时候，也会跳转到此处。

    1. 如设置了tracelimit，则调用tracing(0)给文件`trace_fd`写入0，以确保触发器已关闭。
    2. 如tracemark_fd和trace_fd还没有关闭，则调用`close()`关闭之。
    3. 如设置了enable_events，则调用`event_disable_all()`向文件events/enable写入0。
    4. 如果设置了tracetype但没有设置notrace，则调用`setkernvar`向文件ftrace_enabled写入0。
    5. 如果设置了lockall则调用`munlockall()`解除内存的锁定。
    6. 现代内核版本都高于2.6.28了，不考虑此if分支。
    7. 如果/dev/cpu_dma_latency还是打开状态，则关闭之。
    8. 如果affinity_mask非NULL，则调用`rt_bitmask_free()`释放它指向的内存。
    9. 调用`exit()`返回ret。

##### timerthread线程

1. 如果par->node==-1，则不进行numa相关的设置。

2. 如par->cpu==-1，则用户没有设置亲和性，直接跳过if分支；否则，通过`pthread_setaffinity_np()`设置当前线程的亲和性。

3. 把`par->interval`写入到`interval`。

4. 通过`gettid()`设置`stat->tid`。

5. 把`par->signal`添加进当前线程的信号屏蔽字。

6. 如果`par->mode==MODE_CYCLIC`，则创建一个计时器，当计时器超时的时候会向当前线程发信号`par->signal`。并将interval记录在`tspec.it_interval`。

   ```c
   /* POSIX.1b structure for timer start values and intervals.  */
   struct itimerspec  {
   	struct timespec it_interval;
   	struct timespec it_value;
   } tspec;
   ```

7. 使用setscheduler()设置当前线程的调度算法和优先级。

8. 暂忽略设置smi后要执行的if分支。

9. 如果设置了aligned或secaligned，即线程唤醒的时候在时间上对齐，则执行if分支。

   1. 调用`pthread_barrier_wait()`让所有线程都到达`globalt_barr`再执行。
   2. 在`globalt_barr`和`align_barr`两个内存屏障之间：如果线程编号为0，即`par->tnum==0`，则调用`clock_gettime()`把`par->clock`的时间记录在`globalt`里。更进一步地，如果设置的是`secaligned==1`，则意味着要按秒对齐，则需进一步调整`blobalt`。
   3. 调用`pthread_barrier_wait()`让所有线程都到达`align_barr`再执行。
   4. 所有线程都把0号线程记录下的`globalt`时间记录在变量`now`中。
   5. 如果设置了offset，则须把offset的时间加到now上。

10. 如果即没有设置aligned，也没有设置secaligned，则走else分支，调用`clock_gettime()`把`par->clock`的时间记录到`now`里。

11. 用`next`记录`now + interval`的时间。它代表了线程下次过期的绝对时间。

12. 如果设置了`duration`，则用`stop`记录`now + duration`的时间。

13. 如果`par->mode == MODE_CYCLIC`，则用`timer_settime()`启动进程的计时器。

14. 如果`par->mode == MODE_SYS_ITIMER`，则用`setitimer()`启动系统的计时器。

15. 执行`stat->threadstarted++`表示线程已启动。

16. 当`shutdown`还没有置1的时候，不停进行while循环：

    1. 依据par->mode的值，设置等待的方式。MODE_CYCLIC和MODE_SYS_ITIMER是通过信号等；MODE_CLOCK_NANOSLEEP是进程计时器，通过睡眠等；MODE_SYS_NANOSLEEP是系统计时器，通过睡眠等。
    1. 调用`clock_gettime()`让`now`记录时间。
    1. 如设置了`smi`，则用`stat->smi_count`记录系统管理中断的次数。
    1. 用`diff`记录`now - next`的值。
    1. 分别用`stat->min`和`stat->max`记录`diff`的最小值和最大值。如产生了新的最大值且设置了`refresh_on_max`，则执行`pthread_cond_signal()`唤醒睡眠中的主线程。
    1. 用`stat->avg`记录`diff`的累加和。
    1. 如设置了trigger且`diff > trigger`，则执行`trigger_update()`给结构体`current`的相关字段赋值，`current`会自己转移到链表里的下个元素。
    1. 如果用户设置了程序运行时间duration，且已经达到了用户指定的时间now - stop >= 0，则shutdown++，意为可以关闭程序了。
    1. `stopped`变量是线程是否停止的标志，0表示没有停止，1表示停止。当线程没有停止的时候，如果用户设置了追踪的限制时间tracelimit且diff > tracelimit，则表示进程应在当前线程停止，则会标记相关的变量和文件。
    1. 用`stat->act`记录当前`diff`的值。
    1. 如用户分配了缓冲区`stat->values`或`stat->smis`，则它们的掩码`par->bufmsk`非0，此会把每个diff都记录到缓冲区里。如果diff的数量超过缓冲区的大小，则会覆盖之前的记录。
    1. 如设置了`histogram`，则向代表直方图的数组里记录相应的数据。
    1. `stat->cycles++;`
    1. next += interval；如果是在MODE_CYCLIC模式，可能因为信号或线程而溢出了多次，这些也都需要一并计算进去。
    1. 如果now > next，则next += interval。
    1. 如果用户设置了循环次数par->max_cycles，且当前线程执行了这么多次的循环，则退出当前的while循环。

17. out标志，用于释放相关的资源。

