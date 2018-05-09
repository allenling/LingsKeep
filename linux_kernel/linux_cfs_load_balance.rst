############
load balance
############


linux中调度的时候, load balance的策略, 资料挺少的, 只能尽量通过源码去总结了

需要load balance的话, 比如配置了多核架构了, 比如SMP, 还可以参考其他架构, 比如SMT(Simultaneous Multithreading), SMP, NUMA等等

所以得先了解下计算机的整体结构

.. [1] https://www.ibm.com/developerworks/cn/linux/l-cn-schldom/

.. [2] https://diego.assencio.com/?index=614d73283d49e939ebfb648cfb86819d

.. [3] https://lwn.net/Articles/80911/

.. [4] http://nanxiao.me/linux-kernel-note-60-scheduling-domain/

.. [5] https://help.ubuntu.com/community/ReschedulingInterrupts

.. [6] https://blog.csdn.net/dog250/article/details/6538240

.. [7] http://minnie.tuhs.org/CompArch/Lectures/week01.html

.. [8] https://www.ibm.com/developerworks/cn/aix/library/au-aix-multicore-multiprocessor/index.html

.. [9] http://www.linfo.org/context_switch.html

.. [10] https://blog.csdn.net/hellojoy/article/details/54744231

.. [11] https://blog.csdn.net/zhaihaifei/article/details/51015528

.. [12] http://www.johnchukwuma.com/training/UnderstandingTheLinuxKernel3rdEdition.pdf

.. [13] http://tinylab.org/how-to-calc-load-and-part1/


参考 [1]_ 是总体流程, 挺简单清楚

参考 [2]_ 是 *lscpu* 输出的cpu信息

参考 [3]_ 算是1的英文版, 但是内容会多一点

参考 [4]_ 指出了/proc/sys/kernel/sched_domain目录中有关于cpu domain的配置.

参考 [5]_ 是rescheduling interrupt, 也就是通知其他cpu需要reschedule的中断, 的解释

参考 [6]_ 是对load balance比较详细的讲解, 不过内核是2.6的

参考 [7]_ 是计算机整体的架构, 是一个教程, 一共有6篇, 简单详细易懂, 不过代码主要是java的

参考 [8]_ 是多核SMP的一些知识, 不过讲得是AIX/UNIX的, 只参考个图解

参考 [9]_ context switch的介绍

参考 [10]_ 是寄存器和cpu各级缓存的介绍

参考 [11]_ 是SMP, SMT, CMP的介绍

参考 [12]_ 是Understanding The Linux Kernel这本书的第三版的电子版(应该不算盗版吧, 电子版好像是免费的?), 其中7.5节是关于负载均衡的

参考 [13]_ 也挺详细的

内容包括:

1. 平均负载的计算

2. cpu亲和性

3. 负载均衡的流程

4. 亲和性如何影响负载均衡

cpu的一些科普知识
======================

cpu大概是怎么制造的? 沙子到单晶硅, 然后刻入等等工艺流程

晶振产生的时钟周期, 石英晶体的受电产生机械摆之间的转化, 参考: http://www.cnblogs.com/xd-elegant/p/4125853.html, https://zh.wikipedia.org/wiki/%E6%97%B6%E9%92%9F%E9%A2%91%E7%8E%87

  *CPU的时钟频率通常是由晶体振荡器的频率决定的。首台商业PC，牛郎星8800（由MITS制造）使用了一个时钟频率为2MHz（200万次/秒）的Intel 8080 CPU。第一台IBM PC的时钟频率是4.77MHz（4,772,727次/秒）。1995年，Intel's Pentium 芯片达到了100 MHz （1亿次/秒），到了2002年，最快的CPU：Intel Pentium 4 达到了3GHz（三十亿次/秒，相当于每个周期3.3*10-10秒*

  *截至2007年，CPU性能的提高主要通过流水线，指令集和多核心技术的创新来实现，而不是时钟频率的提高（时钟频率的提高受到了CPU功耗下降的限制)*
  
  --- 来自时钟周期的wiki

四大cpu架构ARM, X86/Atom, MIPS, PowerPC: https://blog.csdn.net/wangjianno2/article/details/52140936

以及PowerPC和x86架构的小小区别和对比   : https://www.zhihu.com/question/22587769, PowerPC是RISC, 精简指令集, X86是CISC, 复杂指令集

  *一个系统，从应用到中间件到数据库再加文件服务器放到一台 power小型机上跑 7X24小时没问题。 换到x86上，至少需要8台机器。还需要做集群容灾。 当然 还是那8台 pc server 合在一起便宜。*
  
  --- 来自PowerPC和x86的知乎答案

cpu读取指令和存储指令的方式: 寄存器, L1, L2, L3缓存等等


计算机的分层
================

下图来自参考 [7]_

.. code-block:: python

    '''
    
                              +---------------+----------------------+------------------+
                              |               |                      |                  |
    user mode                 | User Programs | Application Programs | Systems Programs |
                              |               |                      |                  |
                              +---------------+----------------------+------------------+
                              |                                                         |
                              | libraries                                               |
                              |                                                         |
                              +---------------------------------------------------------+
                              |                                                         |
    kernel mode               | Operating System                                        |
                              |                                                         |
                              +---------------------------------------------------------+
                              |                 |                  |                    |
    hardware                  | Main Memory     | CPU              | Peripheral Devices |
                              |                 |                  |                    |
                              +-----------------+------------------+--------------------+
    
    '''

User Programs       : 用户自己写的程序, 比如你自己写的java/python等等程序

Application Programs: 其他第三方程序, 比如chrome等等

Systems Programs    : 系统自带的工具, 比如bash等等

上面三个都是运行在用户态, 其实我觉得都可以称为user mode application, 区别不大

libraries           : 就是一些基本库, 比如glibc, 包括了OS一些标准接口, 比如PThread     

Operating System    : 就是操作系统了

接下来就是硬件层面  : cpu, 内存, 和外围设备


而硬件层面组件之间怎么连, 就有不同的了, 比如简单点的, 多个cpu共享一个系统总线(System Bus)去读内存(RAM):

下面来自参考 [8]_, 同时, 这也是一个SMP架构和MC(Multi-Core, 多核)架构, 每一个核心可以运行2个线程, 也就是每个核(core)都是SMT.

*在讨论芯片多线程、多核、多处理器环境的设计注意事项之前，我们会简要介绍这类系统。图 1 所述的系统有两个处理器，每个处理器有两个核心，并且每个核心有两个硬件线程。每个核心有一个 L1 缓存和一个 L2 缓存。因此

每个核心可能都拥有自己的 L2 缓存，或者同一个处理器上的核心可能会共享 L2 缓存。同一个核上的硬件线程会共享 L1 和 L2 缓存。*

.. code-block:: python

    '''
    +---------------------------------------+   +---------------------------------------+
    |                                       |   |                                       |
    | processor0                            |   | processor1                            |
    |                                       |   |                                       |
    | +---------------+  +---------------+  |   | +---------------+  +---------------+  |
    | |  core0        |  |  core1        |  |   | |  core0        |  |  core1        |  |
    | |               |  |               |  |   | |               |  |               |  |
    | |  +---------+  |  |  +---------+  |  |   | |  +---------+  |  |  +---------+  |  |
    | |  | thread0 |  |  |  | thread0 |  |  |   | |  | thread0 |  |  |  | thread0 |  |  |
    | |  +---------+  |  |  +---------+  |  |   | |  +---------+  |  |  +---------+  |  |
    | |               |  |               |  |   | |               |  |               |  |
    | |  +---------+  |  |  +---------+  |  |   | |  +---------+  |  |  +---------+  |  |
    | |  | thread1 |  |  |  | thread1 |  |  |   | |  | thread1 |  |  |  | thread1 |  |  |
    | |  +---------+  |  |  +---------+  |  |   | |  +---------+  |  |  +---------+  |  |
    | |               |  |               |  |   | |               |  |               |  |
    | |  +----------+ |  |  +----------+ |  |   | |  +----------+ |  |  +----------+ |  |
    | |  | L1 cache | |  |  | L1 cache | |  |   | |  | L1 cache | |  |  | L1 cache | |  |
    | |  +----------+ |  |  +----------+ |  |   | |  +----------+ |  |  +----------+ |  |
    | |               |  |               |  |   | |               |  |               |  |
    | |  +----------+ |  |  +----------+ |  |   | |  +----------+ |  |  +----------+ |  |
    | |  | L2 cache | |  |  | L2 cache | |  |   | |  | L2 cache | |  |  | L2 cache | |  |
    | |  +----------+ |  |  +----------+ |  |   | |  +----------+ |  |  +----------+ |  |
    | +---------------+  +---------------+  |   | +---------------+  +---------------+  |
    +---------------------------------------+   +---------------------------------------+

                       |                                          |
                       |                                          |

    +-----------------------------------------------------------------------------------+
    |                                                                                   |
    | System Bus                                                                        | <-----访问RAM------> RAM设备
    |                                                                                   |
    +-----------------------------------------------------------------------------------+
    
    '''



寄存器和缓存, 内存的区别, 参考[10]_

Register

中央处理器内的组成部份。寄存器是有限存贮容量的高速存贮部件，它们可用来暂存指令、数据和位址。在中央处理器的控制部件中，包含的寄存器有指令寄存器(IR)和程序计数器(PC)。在中央处理器的算术及逻辑部件中

包含的寄存器有累加器(ACC)

Cache

即高速缓冲存储器，是位于CPU与主内存间的一种容量较小但速度很高的存储器。由于CPU的速度远高于主内存，CPU直接从内存中存取数据要等待一定时间周期,

Cache中保存着CPU刚用过或循环使用的一部分数据，当CPU再次使用该部分数据时可从Cache中直接调用,这样就减少了CPU的等待时间

提高了系统的效率。Cache又分为一级Cache(L1 Cache)和二级Cache(L2 Cache)，L1 Cache集成在CPU内部，L2 Cache早期一般是焊在主板上,现在也都集成在CPU内部，常见的容量有256KB或512KB L2 Cache

内存

内存包含的范围非常广，一般分为只读存储器(ROM), 随机存储器(RAM)和高速缓存存储器(Cache), 一般说的内存则是RAM

Context Switch
=================

参考 [9]_ 和参考 [7]_的第六篇

参考 [7]_第六篇简单一点

简单的说, CPU寄存器(分Program Count和Register, 这里先不区分)中保存指令和地址, 所以切换只需要把寄存器中的指令和地址切换一下就好了

So, what are the steps of a context switch:

1. Get into kernel mode. 内核态才能切换程序

2. Quickly save the old program's registers somewhere, and the address of the next instruction it was about to execute, so they can be restored later.
   
   This storage area is known as a Process Control Block, or PCB, and it is stored in kernel-mode memory somewhere.

   保存当前程序的指令什么的信息

3. Save the mappings in the current page map, also into the PCB.

4. Unmap all of the old program's pages from the page map.

5. Choose a new program to re-start. Find its PCB.

   选择下一个程序去执行, 找到其指令呀什么的信息

6. Re-map the pages of the new program from its PCB.

7. Re-load the registers of the new program from its PCB.

8. Return to the next instruction of the new program, and return to user-mode at the same time.


SMP/NUMA
================

区别就是, SMP是所有的处理单到连到一条总线, 是平等访问, 比如参考 [8]_中的图

而NUMA则是把n个处理单元, 根据一定的策略, 划成m个区域, 每个区域p个处理单元, 每个域都可以是SMP架构

也就是每一个区域都划分有自己的内存, 并且每个域中的p个处理单元可以SMP的形式去访问内存

然后读个域也互相连通, 这样又可以访问到其他域的内存(远端内存)

显然每个域访问自己的内存很快, 如果访问其他区域的内存, 那么就很慢了.

也就是NUMA是把多个SMP连起来, 各个SMP访问其他SMP的内存比较慢

**关于超线程(Hyper-threading):**


  *Hyper-threading
  
  A hyper-threaded chip is a microprocessor that executes several threads of execution at once; it
  
  includes several copies of the internal registers and quickly switches between them. This technology,
  
  which was invented by Intel, allows the processor to exploit the machine cycles to execute another
  
  thread while the current thread is stalled for a memory access. A hyper-threaded physical CPU is seen
  
  by Linux as several different logical CPUs.*
  
  -- 参考12

也就是一个核心能同时运行多个(一般是2个)task, 也就是(可以类比于)并发了, 一般一个核心只能运行一个task, 但是英特尔的cpu

则是可以保存多个(2个一般)的信息(一般保存在寄存器), 然后当一个task等待读取内存返回的时候, 切换到另外一个task, 是不是很像时间驱动(比如epoll)

不过, 不管怎么样, 对于内核来说, 能同时运行多少个task, 就是有多少个cpu(逻辑cpu). 这样对待cpu简单很多

超线程(HT), 也就是多线程(Multi Threading)技术, 实现有两种, 时分多线程(TMD)和同时多线程(SMT)

所以, SMP指的是核心之间被统一平等对待去访问资源(通过一条总线), 而SMT(HT, MT)则是指一个核心中能同时处理多个线程(task)

以上参考 [11]_


所以, 我理解起来就是:

1. 横向和纵向去提升cpu的话, 横向是堆 **物理处理器个数** 或者单个物理处理器的核心数(core), 这时可以使用SMP和NUMA架构
   
2. 纵向提升单个处理器性能, 可以提升核心(core)的赫兹指数或者引入MT技术, 比如SMT等同时处理多个task(线程, 指令)


cpu信息
=============

命令 *lscpu* 会打印出cpu的信息, lscpu输出解释在参考 [2]_

通过命令可以看到物理cpu, 核心等等信息

负载均衡的代价和目的
==========================

目的自然是防止只有一个cpu在跑, 其他cpu围观的情况, 浪费资源

task在cpu上运行的时候, cpu都会缓存一些数据, 所以移动的代价就是缓存失效了

  *但是在这些 CPU 之间进行 Load Balance 是有代价的，比如对处于两个不同物理 CPU 的进程之间进行负载平衡的话，将会使得 Cache 失效。造成效率的下降。而且过多的 Load Balance 会大量占用 CPU 资源。*

  *还有一个要考虑的就是功耗 (Power) 的问题。一个物理 CPU 中的两个 Virtual CPU 各执行一个进程，显然比两个物理 CPU 中的 Virtual CPU 各执行一个进程节省功耗.
  
  因为硬件上可以实现一个物理 CPU 不用时关掉它以节省功耗。*
  
  --- 参考1


NR_CPUS
===========

配置内核最大支持的cpu个数

这个配置是在/arch/x86/Kconfig有解释

当然, kernel编译启动之后的文件/boot/config-xxx, xxx是版本号, 中也有, 比如/boot/config-4.4.0-119-generic

https://elixir.bootlin.com/linux/v4.15/source/arch/x86/Kconfig

.. code-block:: python

    '''

    config SMP
    	bool "Symmetric multi-processing support"
    	---help---
    	  This enables support for systems with more than one CPU. If you have
    	  a system with only one CPU, say N. If you have a system with more
    	  than one CPU, say Y.
    
    	  If you say N here, the kernel will run on uni- and multiprocessor
    	  machines, but will use only one CPU of a multiprocessor machine. If
    	  you say Y here, the kernel will run on many, but not all,
    	  uniprocessor machines. On a uniprocessor machine, the kernel
    	  will run faster if you say N here.

          后面还有一些描述, 先省略

    config MAXSMP
    	bool "Enable Maximum number of SMP Processors and NUMA Nodes"
    	depends on X86_64 && SMP && DEBUG_KERNEL
    	select CPUMASK_OFFSTACK
    	---help---
    	  Enable maximum number of CPUS and NUMA Nodes for this architecture.
    	  If unsure, say N.
    
    
    config NR_CPUS
    	int "Maximum number of CPUs" if SMP && !MAXSMP
    	range 2 8 if SMP && X86_32 && !X86_BIGSMP
    	range 2 64 if SMP && X86_32 && X86_BIGSMP
    	range 2 512 if SMP && !MAXSMP && !CPUMASK_OFFSTACK && X86_64
    	range 2 8192 if SMP && !MAXSMP && CPUMASK_OFFSTACK && X86_64
    	default "1" if !SMP
    	default "8192" if MAXSMP
    	default "32" if SMP && X86_BIGSMP
    	default "8" if SMP && X86_32
    	default "64" if SMP
    	---help---
    	  This allows you to specify the maximum number of CPUs which this
    	  kernel will support.  If CPUMASK_OFFSTACK is enabled, the maximum
    	  supported value is 8192, otherwise the maximum value is 512.  The
    	  minimum value which makes sense is 2.
    
    	  This is purely to save memory - each supported CPU adds
    	  approximately eight kilobytes to the kernel image.
    
    '''

1. SMP: 支持多核架构.
   
   如果你是单核, 设置N, 如果多核, 设置Y.

   多核机器但是设置了N, 那么内核只使用一个cpu.

   如果多个设置了Y, 那么内核运行多个, 但可能不是全部的核心, 这个根据配置决定

   如果你是单核机器, 那么设置N比设置Y跑得快一点


2. MAXSMP: 设置SMP下最多处理器和NUMA节点, 依赖X86_84, SMP, DEBUG_KERNEL这个三个配置, 且都打开(Y)

   如果不确定是否设置该配置, 设置N

3. NR_CPUS: 内核最多支持的cpu个数, 该配置是去决定于SMP打开, 并且MAXSMP为N

   一般的, 如果CPUMASK_OFFSTACK已经打开了, 则最大个数是8192, 否则最大个数就是512

   最小值是2

   其中的range和default就是限制范围和默认值了

   比如range 2 512 if SMP && !MAXSMP && !CPUMASK_OFFSTACK && X86_64都为真, 那么

   NR_CPUS的值范围就是2-512, 然后default中

   default 8 if SMP && X86_32, 也就是如果SMP设置了y, 然后X86_32已经配置, 那么默认就是8,

   同理, 在X86_64下的SMP则就是8, 因为前面的default判断都不过


来看看/arch/x86/configs/x86_64_defconfig中的定义

https://elixir.bootlin.com/linux/v4.15/source/arch/x86/configs/x86_64_defconfig


.. code-block:: c

    // cgroup调度默认是开启的
    CONFIG_CGROUP_SCHED=y

    // SMP默认是开启的
    CONFIG_SMP=y

    // NR_CPUS默认是64
    CONFIG_NR_CPUS=64

    // SMT多线程模式开启的
    CONFIG_SCHED_SMT=y

    // 默认支持NUMA
    CONFIG_NUMA=y


再来看看Ubuntu中的内核配置, 在x86_64架构, 4核机器中, Ubuntu 16.04.3 LTS, 内核版本4.4.0-116-generic下, 文件/boot/config-4.4.0-119-generic就是内核的配置了

.. code-block:: python

    '''
    
    # 这个是自动生成的文件, 也就是内核编译的时候的配置会
    # 打印在这个文件中
    # Automatically generated file; DO NOT EDIT.
    # Linux/x86_64 4.4.0-119-generic Kernel Configuration
    #
    
    CONFIG_64BIT=y
    CONFIG_X86_64=y
    CONFIG_X86=y
    
    CONFIG_MMU=y
    
    CONFIG_SMP=y
    
    # CONFIG_MAXSMP is not set
    CONFIG_NR_CPUS=512
    CONFIG_SCHED_SMT=y
    CONFIG_SCHED_MC=y
    
    CONFIG_CGROUP_SCHED=y
    CONFIG_FAIR_GROUP_SCHED=y
    
    '''


定义了X86_64, SMP, NUMA, 并且NR_CPUS是512个, 开启了MC(MultiCore)和SMT(同时多线程), 已经CGROUP_SCHED和FAIR_GROUP_SCHED(cfs的组调度)


CPU亲和性
=============


负责均衡策略
==================

因为负载均衡迁移一个task是有代价的, 也就是失去cpu缓存, 所以linux给出schedule domain的概念.

负载均衡是针对schedule domain的.

  *In other words, a process is moved from one CPU to another only if the total workload of some group
  in some scheduling domain is significantly lower than the workload of another group in the same scheduling
  domain*
  
  --- 参考12

也就是说, 负载均衡是针对同一个schedule domain的, 失衡的情况是当前schedule domain的某个group的负载和其他group的负载有差距的情况.


schedule domain
=====================

来自参考 [1]_


schedule doamin flag
========================

schedule domain的标志位

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/topology.h#L14

.. code-block:: c

    // 明显, 在多核架构下才会出现
    /*
    * sched-domains (multiprocessor balancing) declarations:
    */
    #ifdef CONFIG_SMP
    
    #define SD_LOAD_BALANCE		0x0001	/* Do load balancing on this domain. */
    #define SD_BALANCE_NEWIDLE	        0x0002	/* Balance when about to become idle */
    #define SD_BALANCE_EXEC		0x0004	/* Balance on exec */
    #define SD_BALANCE_FORK		0x0008	/* Balance on fork, clone */
    #define SD_BALANCE_WAKE		0x0010  /* Balance on wakeup */
    #define SD_WAKE_AFFINE		0x0020	/* Wake task to waking CPU */
    #define SD_ASYM_CPUCAPACITY	        0x0040  /* Groups have different max cpu capacities */
    #define SD_SHARE_CPUCAPACITY	0x0080	/* Domain members share cpu capacity */
    #define SD_SHARE_POWERDOMAIN	0x0100	/* Domain members share power domain */
    #define SD_SHARE_PKG_RESOURCES	0x0200	/* Domain members share cpu pkg resources */
    #define SD_SERIALIZE		0x0400	/* Only a single load balancing instance */
    #define SD_ASYM_PACKING		0x0800  /* Place busy groups earlier in the domain */
    #define SD_PREFER_SIBLING	        0x1000	/* Prefer to place tasks in a sibling domain */
    #define SD_OVERLAP		        0x2000	/* sched_domains of this level overlap */
    #define SD_NUMA			0x4000	/* cross-node balancing */


1. SD_LOAD_BALANCE   : 该sd可以执行负载均衡

2. SD_BALANCE_NEWIDLE: 从busy变成idle

3. SD_BALANCE_EXEC   : 

4. SD_NUMA           : 跨numa节点去负载均衡


cpus_allowed/nr_cpus_allowed
================================

cpus_allowed   : 设置task只能在哪几个cpu上运行, 没现在就是全选咯, 是一个bitmap

nr_cpus_allowed: 上一个参数的个数

这两个参数一般都是互相有关系的, 比如nr_cpus_allowed如果是1, 那么cpus_allowed则只有一个cpu

如果nr_cpus_allowed大于1, 那么cpus_allowed则有多个cpu被设置

cpumask_t的计算参考: https://blog.csdn.net/nirenxiaoxiao/article/details/21462053

在task结构中, 定义了cpus_allowed和nr_cpus_allowed两个字段

.. code-block:: c

    struct task_struct {
        int		nr_cpus_allowed;
        cpumask_t	cpus_allowed;
    }


cpus_allowed是一个cpumask_t结构, 这个结构是位图(bitmap)结构, 并且每一位表示一个cpu

https://elixir.bootlin.com/linux/v4.15/source/include/linux/cpumask.h#L16

.. code-block:: c

    typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;

也就是是一个bitmap结构, 和NR_CPUS有关, DECLARE_BITMAP定宏定义:

https://elixir.bootlin.com/linux/v4.15/source/tools/include/linux/bitmap.h#L10

.. code-block:: c

    #define DECLARE_BITMAP(name,bits) \
    	unsigned long name[BITS_TO_LONGS(bits)]

也就是一个long类型的数组, 长度取决于传入的bits, 也就是NR_CPUS, 而数组的名字则是传入的name, 上面传入的name是bits

https://elixir.bootlin.com/linux/v4.15/source/include/linux/bitops.h#L14

.. code-block:: c

    #define BITS_TO_LONGS(nr)	DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))

BITS_PER_BYTE就是一个字节占几位, 一搬是8, 也就是传入的NR_CPUS可以用多好个long来表示

https://elixir.bootlin.com/linux/v4.15/source/tools/include/linux/kernel.h#L16

.. code-block:: c

    #define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))

根据传入的值, 就是(NR_CPUS + 8 * bytes - 1) / (8*bytes)


也就是, DECLARE_BITMAP声明一个long型的数组, 而数组长度取决于传入的NR_CPUS和long类型的长度, 也就是一个位一个cpu

比如NR_CPUS是24, 表示最多支持24个cpu, 而一个long类型是8字节, 8位一个字节, 那么一共有8 * 8=64位, 那么用一个long类型就可以表示所有的24个cpu了, 也就是

DIV_ROUND_UP = (24 + 64 - 1) / 64 = 1, 所以cpumask的数组长度就是1, 数组的名字是bits, 也就是


.. code-block:: c
   
   typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t

   // 等于

   typedef struct cpumask { long bits[1]; } cpumask_t


**随机选择一个cpu, cpumask_any**

https://elixir.bootlin.com/linux/v4.15/source/include/linux/cpumask.h#L558

.. code-block:: c

    /**
     * cpumask_any - pick a "random" cpu from *srcp
     * @srcp: the input cpumask
     *
     * Returns >= nr_cpu_ids if no cpus set.
     */
    #define cpumask_any(srcp) cpumask_first(srcp)


**cpumask_first**

https://elixir.bootlin.com/linux/v4.15/source/include/linux/cpumask.h#L182

.. code-block:: c

    // NR_CPUS==1, 就是一个单cpu机器
    // 永远返回cpu0
    #if NR_CPUS == 1

    /* Uniprocessor.  Assume all masks are "1". */
    static inline unsigned int cpumask_first(const struct cpumask *srcp)
    {
    	return 0;
    }

    #else

    // 否则, 返回第一个可用bit

    /**
     * cpumask_first - get the first cpu in a cpumask
     * @srcp: the cpumask pointer
     *
     * Returns >= nr_cpu_ids if no cpus set.
     */
    static inline unsigned int cpumask_first(const struct cpumask *srcp)
    {
    	return find_first_bit(cpumask_bits(srcp), nr_cpumask_bits);
    }
    #endif

其中, 传入的nr_cpumask_bits是取决于CPUMASK_OFFSTACK的配置, 不管一般都不配置

.. code-block:: c

    #ifdef CONFIG_CPUMASK_OFFSTACK
    /* Assuming NR_CPUS is huge, a runtime limit is more efficient.  Also,
     * not all bits may be allocated. */
    #define nr_cpumask_bits	nr_cpu_ids
    #else
    #define nr_cpumask_bits	((unsigned int)NR_CPUS)
    #endif

所以, *cpumask_any(&p->cpus_allowed) -> cpumask_first(&p->cpus_allowed) -> find_first_bit(cpumask_bits(&p->cpus_allowed), NR_CPUS)*

而cpumask_bits是返回cpumask的bits数组, 也就是find_first_bit(&p->cpus_allowed->bits, NR_CPUS)

**也就是如果设置了只能在一个cpu上运行, 那么找到cpus_allowed中第一个(唯一一个)cpu**

被唤醒的时候(唤醒和新建)
============================

在epoll中, default_wake_function会调用到try_to_wake_up, 而try_to_wake_up会:

1. 拿到被唤醒task的cpu
   
2. 调用select_task_rq去再分配cpu

3. 如果再分配的cpu不是task中原来的cpu, 设置WF_MIGRATED标志, 表示task被从一个cpu移动到另外一个cpu

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3625

.. code-block:: c

    int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags,
    			  void *key)
    {
    	return try_to_wake_up(curr->private, mode, wake_flags);
    }

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L2063

.. code-block:: c

    static int
    try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
    {
    
        // 拿到当前的
        cpu = task_cpu(p);
        
        // 重新选择cpu
        cpu = select_task_rq(p, p->wake_cpu, SD_BALANCE_WAKE, wake_flags);
        if (task_cpu(p) != cpu) {
            // cpu变了, 设置标志位
            wake_flags |= WF_MIGRATED;
            set_task_cpu(p, cpu);
        }
    
    }


而如果是新建task, 那么在 *_do_fork -> wake_up_new_task* 中, 会重新选一次cpu, 也会调用select_task_rq

**不过, 传入的参数不是SD_BALANCE_WAKE, 而是SD_BALANCE_FORK**


.. code-block:: c

    void wake_up_new_task(struct task_struct *p)
    {
    
    #ifdef CONFIG_SMP
    	/*
    	 * Fork balancing, do it here and not earlier because:
    	 *  - cpus_allowed can change in the fork path
    	 *  - any previously selected CPU might disappear through hotplug
    	 *
    	 * Use __set_task_cpu() to avoid calling sched_class::migrate_task_rq,
    	 * as we're not fully set-up yet.
    	 */
        // 这里!!!!!!!!!!!!!
    	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
    #endif
    
    }


select_task_rq
===================

.. code-block:: c

    /*
     * The caller (fork, wakeup) owns p->pi_lock, ->cpus_allowed is stable.
     */
    static inline
    int select_task_rq(struct task_struct *p, int cpu, int sd_flags, int wake_flags)
    {
    	lockdep_assert_held(&p->pi_lock);
    
        // 如果设置了只能在一个cpu上运行
        // 那么找到cpus_allowed中第一个cpu
        // 否则, 调用调度类的select_task_rq函数
        if (p->nr_cpus_allowed > 1)
    	    cpu = p->sched_class->select_task_rq(p, cpu, sd_flags, wake_flags);
    	else
    	    cpu = cpumask_any(&p->cpus_allowed);
    
    	/*
    	 * In order not to call set_task_cpu() on a blocking task we need
    	 * to rely on ttwu() to place the task on a valid ->cpus_allowed
    	 * CPU.
    	 *
    	 * Since this is common to all placement strategies, this lives here.
    	 *
    	 * [ this allows ->select_task() to simply return task_cpu(p) and
    	 *   not worry about this generic constraint ]
    	 */
    	if (unlikely(!cpumask_test_cpu(cpu, &p->cpus_allowed) ||
    		     !cpu_online(cpu)))
    	    cpu = select_fallback_rq(task_cpu(p), p);
    
    	return cpu;
    }

所以, 如果不限制只在一个cpu上运行的话, 调用选一个

select_task_rq_fair
====================

cfs中, select_task_rq被指向select_task_rq_fair

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L6318

.. code-block:: c

    /*
     * select_task_rq_fair: Select target runqueue for the waking task in domains
     * that have the 'sd_flag' flag set. In practice, this is SD_BALANCE_WAKE,
     * SD_BALANCE_FORK, or SD_BALANCE_EXEC.
     *
     * Balances load by selecting the idlest cpu in the idlest group, or under
     * certain conditions an idle sibling cpu if the domain has SD_WAKE_AFFINE set.
     *
     * Returns the target cpu number.
     *
     * preempt must be disabled.
     */
    static int
    select_task_rq_fair(struct task_struct *p, int prev_cpu, int sd_flag, int wake_flags)
    {
    	struct sched_domain *tmp, *affine_sd = NULL, *sd = NULL;
    	int cpu = smp_processor_id();
    	int new_cpu = prev_cpu;
    	int want_affine = 0;
    	int sync = wake_flags & WF_SYNC;
    
    	if (sd_flag & SD_BALANCE_WAKE) {
    	    record_wakee(p);
    	        want_affine = !wake_wide(p) && !wake_cap(p, cpu, prev_cpu)
    			      && cpumask_test_cpu(cpu, &p->cpus_allowed);
    	}
    
    	rcu_read_lock();
    	for_each_domain(cpu, tmp) {
    	    if (!(tmp->flags & SD_LOAD_BALANCE))
    	    	break;
    
    	    /*
    	     * If both cpu and prev_cpu are part of this domain,
    	     * cpu is a valid SD_WAKE_AFFINE target.
    	     */
    	    if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
    	        cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
    	    	affine_sd = tmp;
    	    	break;
    	    }
    
    	    if (tmp->flags & sd_flag)
    	    	sd = tmp;
    	    else if (!want_affine)
    	    	break;
    	}
    
    	if (affine_sd) {
    	    sd = NULL; /* Prefer wake_affine over balance flags */
    	    if (cpu == prev_cpu)
    	    	goto pick_cpu;
    
    	    if (wake_affine(affine_sd, p, prev_cpu, sync))
    	        new_cpu = cpu;
    	}
    
    	if (sd && !(sd_flag & SD_BALANCE_FORK)) {
    	    /*
    	     * We're going to need the task's util for capacity_spare_wake
    	     * in find_idlest_group. Sync it up to prev_cpu's
    	     * last_update_time.
    	     */
    	    sync_entity_load_avg(&p->se);
    	}
    
    	if (!sd) {
    pick_cpu:
    	    if (sd_flag & SD_BALANCE_WAKE) /* XXX always ? */
                 new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);
    
    	} else {
            new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);
    	}
    	rcu_read_unlock();
    
    	return new_cpu;
    }

注释上说明了流程: 找到找到其中最空闲组的最空闲的cpu, 如果SD_WAKE_AFFINE标志位被设置, 则找到兄弟(附近)空闲的cpu

周期性负载均衡
=================

在scheduler_tick中, 每一个时钟周期(注意的是, 不是cpu周期, 是内核定义的时钟周期, 一般是1ms)都回去判断和处理负载均衡

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3012

.. code-block:: c

    void scheduler_tick(void)
    {
    	int cpu = smp_processor_id();
    	struct rq *rq = cpu_rq(cpu);
    
    #ifdef CONFIG_SMP
    	rq->idle_balance = idle_cpu(cpu);
        // 如果是多核系统, 那么发送软中断去处理负载均衡
    	trigger_load_balance(rq);
    #endif
    	rq_last_tick_reset(rq);
    }


trigger_load_balance是发送一个软中断, 这个软中断叫SCHED_SOFTIRQ, 然后该软中断的处理函数回去做负载均衡的

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9389

.. code-block:: c

    /*
     * Trigger the SCHED_SOFTIRQ if it is time to do periodic load balancing.
     */
    void trigger_load_balance(struct rq *rq)
    {
    	/* Don't need to rebalance while attached to NULL domain */
    	if (unlikely(on_null_domain(rq)))
    		return;
    
        // 如果打到了rq上下一次负载均衡的时间点
    	if (time_after_eq(jiffies, rq->next_balance))
            // 发起软中断
    	    raise_softirq(SCHED_SOFTIRQ);
    #ifdef CONFIG_NO_HZ_COMMON
    	if (nohz_kick_needed(rq))
    	    nohz_balancer_kick();
    #endif
    }


软中断的处理函数是run_rebalance_domains

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9970

.. code-block:: c

    #ifdef CONFIG_SMP
    	open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
    
    #ifdef CONFIG_NO_HZ_COMMON
    	nohz.next_balance = jiffies;
    	zalloc_cpumask_var(&nohz.idle_cpus_mask, GFP_NOWAIT);
    #endif
    #endif /* SMP */


run_rebalance_domains
=========================

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9368

.. code-block:: c

    static __latent_entropy void run_rebalance_domains(struct softirq_action *h)
    {
    	struct rq *this_rq = this_rq();
    	enum cpu_idle_type idle = this_rq->idle_balance ?
    						CPU_IDLE : CPU_NOT_IDLE;
    
    	/*
    	 * If this cpu has a pending nohz_balance_kick, then do the
    	 * balancing on behalf of the other idle cpus whose ticks are
    	 * stopped. Do nohz_idle_balance *before* rebalance_domains to
    	 * give the idle cpus a chance to load balance. Else we may
    	 * load balance only within the local sched_domain hierarchy
    	 * and abort nohz_idle_balance altogether if we pull some load.
    	 */
    	nohz_idle_balance(this_rq, idle);
    	rebalance_domains(this_rq, idle);
    }


功能主要在rebalance_domains中

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9110


schedule/__schedule中的负载均衡
=================================

如果选择下一个task运行不存在, 并且当前rq是空闲的话, 就得从其他cpu拿一个task来运行了

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3287

.. code-block:: c

    static void __sched notrace __schedule(bool preempt)
    {
    
        // 这里必然会选择一个task来运行
        // 所以, 这里会去从其他cpu拉取一个task过来
        next = pick_next_task(rq, prev, &rf);
    
    }


**pick_next_task调用cfs对应的函数去拉取其他cpu的task**

.. code-block:: c

    static inline struct task_struct *
    pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
    {
    	const struct sched_class *class;
    	struct task_struct *p;
    
    	if (likely((prev->sched_class == &idle_sched_class ||
    		    prev->sched_class == &fair_sched_class) &&
    		   rq->nr_running == rq->cfs.h_nr_running)) {
    
                // 调用cfs中的pick_next_task
    		p = fair_sched_class.pick_next_task(rq, prev, rf);
                // 需要重新拉取
    		if (unlikely(p == RETRY_TASK))
                    // 重新拉取
    		    goto again;
    
    		/* Assumes fair_sched_class->next == idle_sched_class */
    		if (unlikely(!p))
    			p = idle_sched_class.pick_next_task(rq, prev, rf);
    
    		return p;
    	}
    
    again:
        // cfs没有下一个task了
        // 那么从其他调度类拿到下一个task
    	for_each_class(class) {
    	    p = class->pick_next_task(rq, prev, rf);
    	    if (p) {
    	    	if (unlikely(p == RETRY_TASK))
    	    	    goto again;
    	    	return p;
    	    }
    	}
    
    	/* The idle class should always have a runnable task: */
    	BUG();
    }

pick_next_task_fair
=======================

这里, 如果拿不到下一个task, 那么从其他cpu拿一个


.. code-block:: c

    static struct task_struct *
    pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
    {
    
    idle:
        // 拿不到下一个task了, 调用idle_balance去拉取一个task
    	new_tasks = idle_balance(rq, rf);
    
    	/*
    	 * Because idle_balance() releases (and re-acquires) rq->lock, it is
    	 * possible for any higher priority task to appear. In that case we
    	 * must re-start the pick_next_entity() loop.
    	 */
    	if (new_tasks < 0)
    		return RETRY_TASK;
    
    	if (new_tasks > 0)
    		goto again;
    
    	return NULL;
    }


idle_balance
===============

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L3900

.. code-block:: c

    /*
     * idle_balance is called by schedule() if this_cpu is about to become
     * idle. Attempts to pull tasks from other CPUs.
     */
    static int idle_balance(struct rq *this_rq, struct rq_flags *rf)
    {
    
    }

