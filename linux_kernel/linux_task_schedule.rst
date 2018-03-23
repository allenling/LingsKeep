########################
内核中的task结构以及调度
########################

参考:

.. [1] https://tampub.uta.fi/bitstream/handle/10024/96864/GRADU-1428493916.pdf
 
.. [2] https://www.cnblogs.com/wangzahngjun/p/4977425.html (slab的简单解释)

.. [3] https://www.ibm.com/developerworks/cn/linux/l-cn-slub/ (slub的简单解释)

.. [4] https://zhuanlan.zhihu.com/p/33389178

.. [5] https://zhuanlan.zhihu.com/p/33461281

.. [6] http://www.linuxjournal.com/magazine/completely-fair-scheduler

.. [7] http://blog.csdn.net/farmwang/article/details/70174192

.. [8] https://www.ibm.com/developerworks/library/l-linux-smp/

.. [9] https://elinux.org/images/d/dc/Elc2013_Na.pdf

.. [10] https://www.jianshu.com/p/81233f3c2c14

.. [11] http://colobu.com/2016/04/14/Amdahl-s-Law/

.. [12] https://www.ibm.com/developerworks/cn/linux/l-affinity.html

.. [13] http://people.cs.ksu.edu/~gud/docs/ppt/scheduler.pdf

.. [14] https://www.ibm.com/developerworks/library/l-scheduler/index.html

**参考1, 6, 9是主要参考, 包括linux的调度历史, O(1)调度以及CFS的概念和源码解释**

参考 [4]_是关于linux调度的一个简介, 参考 [5]_是O(1)调度的解释

参考6是O(1)调度和CFS(Completely Fair Scheduler)的介绍

参考7是简单中文解释try_to_wake_up函数, 一个主要唤醒函数.

参考8是linux smp的一些历史, 简单来说, 单核cpu性能打到物理限制之后, 就需要横向拓展, 增加cpu个数, 也就是多核架构.

参考9中一些图示对linux的调度解释更清晰, 并提出一个改进方案, 改进cfs的多核下load balance, 是得多核下cpu更使用公平 

参考9的方案称为DWRR(Distributed Weighted Round-Robun), 看起来(里面有测试)比cfs在大多数情况好

参考10是关于smp, numa, mpp三个架构的简单中文介绍

参考11是关于阿姆达尔定律的简单解释

参考12是关于cpu亲和性的解释

参考13, 14是O(1)调度器的流程介绍

2.6.23至今(4.15)linux已经是CFS调度为主了

**关键点: 定时的抢占流程, 陷入等待流程(以ep_wait为例子), 以及被唤醒的流程, cfs, smp下的load balance**

SMP架构
=============

SMP, Symmetric Multiprocessing, 对称多处理, 也就是多核处理器架构, 多个处理器共享全局内存地址.
  
  *在SMP中所有的处理器都是对等的, 它们通过总线连接共享同一块物理内存，这也就导致了系统中所有资源(CPU、内存、I/O等)都是共享的，当我们打开服务器的背板盖，如果发现有多个cpu的槽位，但是却连接到同一个内存插槽的位置，那一般就是smp架构的服务器, 日常中常见的pc啊，笔记本啊，手机还有一些老的服务器都是这个架构，其架构简单，但是拓展性能非常差*
  
  --- 参考10

  *从linux 上也能看到: ls /sys/devices/system/node/# 如果只看到一个node0 那就是smp架构*
  
  --- 参考10


以下摘抄翻译自参考 [8]_

*An SMP architecture is simply one where two or more identical processors connect to one another through a shared memory.*

SMP架构中, 两个或多个同样的处理器通过一块共享内存彼此连接。每个处理器可同等地访问共享内存（具有相同的内存空间访问延迟）

*Unfortunately, because not all of the problem can be parallelized and there's overhead in managing the processors, the speedup is quite a bit less*

阿姆达尔定律(Amdahl's law), 也即是增加核心数对性能提升的公式, 也就是增加核心数对性能提升并不是线性的. **因为并不是所有的任务都能并行化(比如io)并且多核的管理开销也会上升**.

*Contrast this with the Non-Uniform Memory Access (NUMA) architecture. For example, each processor has its own memory but also access to shared memory with a different access latency.*

SMP一般和NUMA架构比较, NUMA是每个cpu都有自己的内存地址(通过地址划分), 而不是共享的.

*To make use of SMP with Linux on SMP-capable hardware, the kernel must be properly configured. The CONFIG_SMP option must be enabled during kernel configuration to make the kernel SMP aware*

linux kernel支持SMP架构, 需要编译的时候带上CONFIG_SMP选项

  *当处理器频率达到其极限时，一种流行的提高性能的方法是添加更多的处理器。在早期，这就意味着将更多的处理器添加到主板上，或将多个独立计算机集群到一起。现在，芯片级多处理能够在单个芯片上提供更多的 CPU，由于减少了内存延迟，因而可获得更高的性能。
  您会发现 SMP 系统不仅存在于服务器中，还存在于桌面上，特别是在引入虚拟化以后。跟大多数先进技术一样，Linux 提供 SMP 支持。内核负责完成可用 CPU 间的负载优化（从线程到虚拟化操作系统）。惟一要做的就是确保应用程序可被充分地多线程化以便使用 SMP 的能力。*
  
  --- 参考8的结束语

还有一个叫AMP, Asymmetric MultiProcessor, 和SMP相反, 是非对称多处理.

AMP参考:

1. http://www.electronicdesign.com/digital-ics/symmetric-multiprocessing-vs-asymmetric-processing

2. https://www.embedded.com/design/mcus-processors-and-socs/4429496/Multicore-basics

NUMA/MMP
===========

下面来自参考 [10]_

NUMA, Non-Uniform Memory Access, 非均匀访问存储模型, 如果说smp 相当于多个cpu 连接一个内存池导致请求经常发生冲突的话，numa 就是将cpu的资源分开, 以node为单位进行切割,

每个node里有着独有的core, memory等资源, 这也就导致了cpu在性能使用上的提升. 但是同样存在问题就是2个node 之间的资源交互非常慢,

当cpu增多的情况下，性能提升的幅度并不是很高。所以可以看到很多明明有很多core的服务器却只有2个node区

MPP, Massive Parallel Processing, 这个其实可以理解为刀片服务器，每个刀扇里的都是一台独立的smp架构服务器，且每个刀扇之间均有高性能的网络设备进行交互，保证了smp服务器之间的数据传输性能。相比numa 来说更适合大规模的计算，唯一不足的是，当其中的smp 节点增多的情况下，与之对应的计算管理系统也需要相对应的提高。

阿姆达尔定律
===============

主要摘抄自参考 [11]_

*1967年计算机体系结构专家吉恩.阿姆达尔提出过一个定律阿姆达尔定律，说：在并行计算中用多处理器的应用加速受限于程序所需的串行时间百分比。譬如说，你的程序50%是串行的，其他一半可以并行，那么，最大的加速比就是2。不管你用多少处理器并行，这个加速比不可能提高。在这种情况下，改进串行算法可能比多核处理器并行更有效.*

*阿姆达尔定律是固定负载（计算总量不变时）时的量化标准*

**阿姆达尔定律总结起来: 在固定负载下, 也就是不管多少核心, 并行化的提升就依赖于不能并行化的那部分!!**

*阿姆达尔定律的结论让人沮丧，但到了20世纪80年代晚期，Sandia国家实验室的科学家们在对具有1024个处理器的超立方体结构上观察到了3个实际应用程序随着处理器的增加发生线性加速的现象，科学家John L. Gustafson基于此实验数据在1988年提出了一个新的计算加速系数的公式*

*阿姆达尔定律的问题出在它的前提过于理想化。因为并行算法通常能处理比串行算法更大规模的问题，即使算法仍然存在着串行部分，但由于问题规模的不断扩大，往往会导致算法中串行部分所占比例的持续减少*

**感觉提升的原因的重点在于: 但由于问题规模的不断扩大，往往会导致算法中串行部分所占比例的持续减少**, 其实还是逃不开阿姆达尔中的结论, 也就是提升受限于不能串行化部分. 也可以说

串行部分占比越少, 提升越大, 感觉两个结论都差不多意思.

cpu亲和性
============

摘抄自参考 [12]_

*简单地说，CPU 亲和性（affinity） 就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。Linux 内核进程调度器天生就具有被称为 软 CPU 亲和性（affinity） 的特性，这意味着进程通常不会在处理器之间频繁迁移。这种状态正是我们希望的，因为进程迁移的频率小就意味着产生的负载小。*

*其中与 亲和性（affinity）相关度最高的是 cpus_allowed 位掩码。这个位掩码由 n 位组成，与系统中的 n 个逻辑处理器一一对应。 具有 4 个物理 CPU 的系统可以有 4 位。如果这些 CPU 都启用了超线程，那么这个系统就有一个 8 位的位掩码。
如果为给定的进程设置了给定的位，那么这个进程就可以在相关的 CPU 上运行。因此，如果一个进程可以在任何 CPU 上运行，并且能够根据需要在处理器之间进行迁移，那么位掩码就全是 1。实际上，这就是 Linux 中进程的缺省状态。*

也就是把task绑定到指定的cpu上, 因为task切换会减少cpu缓存的命中.

可以设置多个吗?

比如4核的机器我亲和其中的两个, 但是设置两个亲和度的话, 这样task还是会被调度到另外一个cpu, 依然有

调度发生, 这样的话就丧失了设置亲和度的优势了. 所以感觉(推测)亲和度一般指定其中一个cpu.

调度单位
=============

内核的调度单位是task, 无论是进程还是线程, 都会映射到task结构中, 也就是lwp(Light Weight Process).

而linux的线程的实现是glibc下的nptl实现的, 具体参考: glibc_nptl.rst

KThread
---------------

KThread是内核线程.

内线线程和lwp有区别是两个意思: lwp(task)是内核的调度单位, 内核线程也是对应一个task结构, 只是内核线程只能由内核去管理, 用户是终止不了的.

所以KThread被称为内核运行线程可能更好点, 用来做后台基础任务的, 比如定时刷盘(flush)等等.

https://elixir.bootlin.com/linux/v4.15/source/include/linux/kthread.h

.. code-block:: c

    struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
    					   void *data,
    					   int node,
    					   const char namefmt[], ...);

    struct task_struct *kthread_create_on_cpu(int (*threadfn)(void *data),
    					  void *data,
    					  unsigned int cpu,
    					  const char *namefmt);   
    
看到kthread_create_on_node和kthread_create_on_cpu返回的依然是task结构

可以使用 *ps -fax* 命令查看内核启动的线程



调度器和其功能
================

  *The part of the kernel, which is responsible for granting CPU time to tasks, is called process scheduler.*
  
  -- 参考1

调度器负责把cpu时间分配到(多个)task上

参考 [6]_

*The scheduler is one of the most important components of any OS. Implementing a scheduling algorithm is difficult for a couple reasons.*

调度器实现非常困难了

*First, an acceptable algorithm has to allocate CPU time such that higher-priority tasks (for example, interactive applications like a Web browser) are given preference over low-priority tasks (for example, non-interactive batch processes like program compilation)*

首先, 必须保证高优先级的任务运行时间比低优先级的任务多

*At the same time, the scheduler must protect against low-priority process starvation. In other words, low-priority processes must be allowed to run eventually, regardless of how many high-priority processes are vying for CPU time.*

同时, 必须保证低优先级的任务一定会运行, 不然低优先级任务就卡主了呀.


2.4以及之前的O(n)调度
=====================

基本就是遍历了, 这部分略过吧


O(1)的调度策略
====================

  *global runqueue 带来的性能问题其实还可以忍受，毕竟只是在 dequeue 的过程需要加锁；接下来这个问题，就很要命 —— 2.4 scheduler 的时间复杂度是 O(N)。*
  
  --- 参考5
  
这里的global是因为之前是单核系统, 所以只有一个runqueue, 然后在多核情况下(smp), 对runqueue的操作只能是加锁串行化了
  
  *2.4 scheduler 的时间复杂度是 O(N)。我们知道，现代操作系统都能运行成千上万个进程，O(N) 的算法意味着每次调度时，对于当前执行完的 process，需要把所有在 expired queue 中的 process 过一遍，找到合适的位置插入*
  
  --- 参考5
  
  *对于那些对2.4 scheduler 不太了解的同学咱们多说两句：2.4 scheduler 维护两个 queue：runqueue 和 expired queue。两个 queue 都永远保持有序，一个 process 用完时间片，就会被插入 expired queue；当 runqueue 为空时，只需要把 runqueue 和 expired queue 交换一下即可。*
  
  --- 参考5

参考 [5]_原文的流程是:

1. 在 active bitarray 里，寻找 left-most bit 的位置 x。

2. 在 active priority array（APA）中，找到对应队列 APA[x]。

3. 从 APA[x] 中 dequeue 一个 process，dequeue 后，如果 APA[x] 的 queue 为空，那么将 active bitarray 里第 x bit置为 0。

4. 对于当前执行完的 process，重新计算其 priority，然后 enqueue 到 expired priority array（EPA）相应的队里 EPA[priority]。

5. 如果 priority 在 expired bitarray 里对应的 bit 为 0，将其置 1。

6. 如果 active bitarray 全为零，将 active bitarray 和 expired bitarray 交换一下。


下面代码来自参考 [1]_

.. code-block:: c

    struct runqueue {
     unsigned long nr_running; /* number of runnable tasks */
     // 其他代码省略
     struct prio_array *active; /* pointer to the active priority array */
     struct prio_array *expired; /* pointer to the expired priority array */
     struct prio_array arrays[2]; /* the actual priority arrays */
    }


所以每一个runqueue都有自己的active queue和expired queue, 然后使用active指向arrays这个数组中的一个, expired指向另外一个元素

交换active和expired则是交换指针.

而prio_array的结构如下:

.. code-block:: c

    struct prio_array {
     int nr_active; /* number of tasks */
     unsigned long bitmap[BITMAP_SIZE]; /* priority bitmap */
     struct list_head queue[MAX_PRIO]; /* priority queues */
    };

每一个prio_array都有bitmap以及对应的task数组, 所以有


.. code-block:: python

    '''
    
    
    runqueue +--------------+ active  +------------>----->>>>>---+
             |                                                   |
             |                                                   |
             +------------- + expired +->->-+                    |
             |                              |                    |
             |                              |                    |
             |                              |                    |
             +-------------arrays ------> prio_array -->--->--prio_array
                                                                 |
                                                                 |
                                                                 |
                                                                 +-------+ bitmap (这里是140个优先级)
                                                                         |
                                                                         |
                                                                         + queue  (queue中的每一个元素都是一个task链表, 获取下一个task, 是fifo获取)
    
    
    '''


重新计算优先级和timeslice
----------------------------

task的优先级计算是动态计算, 也就是当一个task用完timeslice之后, 会重新计算其优先级和其timeslice, 将其移动(append)到新优先级的queue中.

计算的时候根据其睡眠时间去判断是否是io密集, 如果是, 提升其优先级.

  *When a task on the active runqueue uses all of its time slice, it's moved to the expired runqueue. During the move, its time slice is recalculated (and so is its priority; more on this later)*
  
  --- 参考14


O(1)调度器的问题
-------------------

  *However, a seemingly flawless design had one great issue built into it from the beginning. Overwhelmingly complex heuristics were used to mark a task as interactive or IO-bound. The
  algorithm tried to identify interactive processes by analysing the average sleep time (waiting for input) and the scheduler gave a priority bonus to such tasks for better throughput and user
  experience. The calculations were so complex and error prone that they made processes behave not accordingly to their assumed interactivity level from time to time. Furthermore, people were
  complaining about rather intricate codebase*
  
  --- 参考1

  *Tasks are determined to be I/O-bound or CPU-bound based on an interactivity heuristic. A task's interactiveness metric is calculated based on how much time the task executes compared to how much time it sleeps. Note that because I/O tasks schedule I/O and then wait, an I/O-bound task spends more time sleeping and waiting for I/O completion. This increases its interactive metric.*
  
  -- 参考14

* Slow response time
  Frequent time slice allocation

* Throughput fall
  Excessive switching overhead

* None fair condition(优先级之间timeslice差别会很大, 而cfs使用load weight, 结果差别不大)
  Nice 0 (100ms), Nice 1(95ms) => 5%
  Nice 18(10ms), Nice 19(5ms) => 50% 

上面三点来自参考 [9]_

简单来说, O(1)调度器会根据一个task的平均睡眠时间去判断该task是否是io密集型的task, 如果是, 则提升优先级(gave a priority bonus to such tasks for better throughput and user experience)

但是这个计算过程太复杂, 不够鲁棒.

  *The main issue with this algorithm is the complex heuristics used to mark a task as interactive or non-interactive. The algorithm tries to identify interactive processes by analyzing average sleep time (the amount of time the process spends waiting for input). Processes that sleep for long periods of time probably are waiting for user input, so the scheduler assumes they're interactive. The scheduler gives a priority bonus to interactive tasks (for better throughput) while penalizing non-interactive tasks by lowering their priorities. All the calculations to determine the interactivity of tasks are complex and subject to potential miscalculations, causing non-interactive behavior from an interactive process.*
  
  --- 参考6, 说计算task是否是io密集是基于平均睡眠时间, 睡眠时间的计算以及计算timeslice很复杂, 也容易出现错误判断.

下面是参考 [13]_中关于动态计算优先级, 判断task是否是io密集任务的流程

* Penalty (addition) for CPU bound tasks and reward (subtraction) for I/O bound tasks [-5, 5]

* *p->sleep_avg*: average amount of time a task sleeps vs.average amount of time task uses CPU.
   p->sleep_avg += sleep_time
   p->sleep_avg -= run_time

* Higher sleep_avg –> more I/O bound the task -> more reward. And vice versa.

所以就是, sleep_avg这个属性计算之后, sleep_avg更大的, 优先级更高

关于睡眠时间

  *Earned when a task sleeps for a 'long' time, Spent when a task runs for a 'long' time*
  
  --- 参考13

也就是睡眠了一段时间, 比如10ms, 就加上10ms, 一直运行了5ms, 然后进入睡眠, 就减去这5ms, 就是上面sleep_avg的操作

所以:

1. O(1)是没有抢占的!!!因为它是找bitmap中第一个被置为1的优先级, 去运行该优先级下的runqueue

2. O(1)根据task的平均睡眠时间去判断task是否是io密集, 然后这个过程计算复杂且容易出错


针对O(1)的交互性优化
==========================

参考 [1]_

看起来O(1)对于交互性任务还是不够友好, Con Kolivas这个哥们就自己去优化(文章说他是一个麻醉师...), 对O(1)进行了针对交互性程序优化, 然后搞出来"The Staircase Scheduler"

然后针对CFS, 弄出了BFS(Brain Fuck Scheduler). 更多查看参考 [1]_

CFS
=====

现在O(1)的调度策略被一个更强调公平的调度策略取代了, 称为Completely Fair Scheduler.

CFS总结起来就是

  *According to Ingo Molnar, the author of the CFS, its core design can be summed up in single sentence: “CFS basically models an 'ideal, precise multitasking CPU' on real hardware.”*
  
  --- 参考6和参考1

也就是CFS模拟一个理想的, 精确的多任务处理器...

理想的和精确的例子:

  *For example, given 10 milliseconds, if there were two batch tasks executing, a normal scheduler would offer them 5 milliseconds with 100% CPU power each. An ideal processor would run them
  both simultaneously for 10 milliseconds with each getting 50% CPU. The later model is called perfect multitasking.*
  
  --- 参考1

理想的(单)处理器会同时运行两个任务, 让他们各自使用50%的cpu.
  
  *This is of course impractical – it is physically impossible to run any more than one execution flow on a single processor(core). So, CFS tries to mimic perfectly fair scheduling. Rather than simply
  assign a timeslice to a process, the scheduler calculates how long a task should run as a function of the total number of currently runnable processes*
  
  --- 参考1

但处理器当然不能同时运行多个任务, 所以cfs只是模拟. 也就是cfs不是简单地根据task数量去划分task的timeslice, 而是task的timeslice是根据当前可运行的所有的task计算出来的.

也就是, 两个任务a, b, 10ms的cpu时间, 一般的调度器会让a完全占据前面5ms,然后后面5ms给b, 也就是, 而所谓理想的精确的调度器, 则动态分配timeslice给a, b, 在10ms中不断切换, 让a, b **最终** 公平地运行.

参考 [6]_中给出的例子更清楚点, 也就是比如4个task, a, b, c, d, 一般的调度器会平均分配每一个task占据25%的cpu时间, 然后每一个25%都是task独占着时间片段, 其他任务必须等待.

也就是第一个25%时间运行a, 那么b, c, d都会等待, 而cfs则不是根据数量去平均划分cpu时间, 而是根据每一个task的优先级去划分每一个task应得的timeslice.

然后在某个时间点, **另外一个task会抢占掉当前task**, 然后被抢占的task重新计算timeslice, 最终, 每一个task都能公平的使用cpu.

**关键单在于根据优先级计算timeslice, 然后允许抢占, 这样a, b, c, d则互相抢占, 达到"公平地"使用cpu的目的.**


CFS调度的周期/策略
====================

下面的参考都出自参考 [1]_, 基本上下面就是翻译了.

正常调度, 注意是正常调度, 而不是所有的让出cpu的行为, 发生是每一个钟周期执行的, 内核中时钟周期是1/1000秒(1ms), 其他主动让出cpu的行为, 比如sleep/select等操作主动让出cpu, 也需要调度器

去决定下一个任务是哪一个.

但是, 每个时间周期内核都会去判断是否需要切换当前的task. 如果不需要切换task, 那么当前task则会运行下去/

task运行的时间称为时间片段, timeslice. 如果task一直运行直到时钟中断, 那么task就完全利用了它的timeslice, 否则不能完全利用timeslice.

*Preemptions are caused by timer interrupts. A timer interrupt is basically a clock tick inside the kernel, the clock ticks 1000 times a second;*

*When an interrupt happens, scheduler has to decide whether to grant CPU to some other process and, if so, to which one. The amount of time a process gets to run is called
 timeslice of a process.*
  

task分类型, 分为cpu密集和io密集, 显然io密集类型的task不是总能完全利用timeslice, 因为它会主动去等待io有发生, 而cpu密集型则总是完全利用. 

一个task不是严格区分类型的, 有可能某个时候是io密集, 某个时候是cpu密集. 调度器的责任则是平衡两种类型的task, 保证每一个task都能有足够的时间片段去运行, 确保cpu的最大利用率.

*A scheduling policy in the system has to balance between the two types of processes, and to make sure that every task gets enough execution resources, with no visible effect on the performance of
other jobs*


为了最大化cpu的利用率, 同时保证task能快速响应, linux是让cpu密集型task运行时间更长, 但是频率(运行次数)不高, 而io类型的task则是运行时间很短, 但是运行次数很多.

这就是所谓的load balance.

*To maximize CPU utilization and to guarantee fast response times, Linux tends to provide non-interactive processes with longer “uninterrupted” slices in a row, but to run them less
frequently. I/O bound tasks, in turn, possess the processor more often, but for shorter periods of time.*


----

下面是代码
===============

task调度相关的属性
======================

.. code-block:: c

    struct task_struct {
    
        // 下面4个是优先级相关
        int prio, static_prio, normal_prio;
        unsigned int rt_priority;
        // 下面3个是调度类, 调度实体和实时任务调度实体
        const struct sched_class *sched_class;
        struct sched_entity se;
        struct sched_rt_entity rt;
        // 调度策略
        unsigned int policy;
        // cpu亲和度
        cpumask_t cpus_allowed;
    
    };

其中调度类操作的是调度实体, 也就是调度实体带的数据不一定是task(一般是task)

可以对比epoll中提到的wait_queue和wait_queue_entry一起理解

调度策略属性/cpu亲和度
===========================

task结构中的policy属性表示task调度的策略, cpus_allowed表示cpu的亲和度的掩码

.. code-block:: c

    unsigned int policy;
    cpumask_t cpus_allowed;

调度策略定义

https://elixir.bootlin.com/linux/v4.15/source/include/uapi/linux/sched.h#L42

.. code-block:: c

    /*
     * Scheduling policies
     */
    #define SCHED_NORMAL		0
    #define SCHED_FIFO		        1
    #define SCHED_RR		        2
    #define SCHED_BATCH		        3
    /* SCHED_ISO: reserved but not implemented yet */
    #define SCHED_IDLE		        5
    #define SCHED_DEADLINE		6


调度类和调度策略并不是强制一一对应关系

  *The kernel decides, which tasks go to which scheduling classes based on their scheduling policy (SCHED_\*) and calls the corresponding functions. Processes under SCHED_NORMAL,
  
  SCHED_BATCH and SCHED_IDLE are mapped to fair_sched_class, provided by CFS. SCHED_RR and SCHED_FIFO associate with rt_sched_class, real-time scheduler*
  
  -- 参考1

也就是说

1. SCHED_RR和SCHED_FIFO的调度类是实时任务调度类
   
2. SCHED_NORMAL说明是一般任务, 使用cfs的调度类, 而SCHED_BATCH和SCHED_IDLE也是用cfs


优先级
==========

参考 [1]_

task中的优先级变量有4个

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L520

.. code-block:: c

    struct task_struct {
        int			 prio;
        int			 static_prio;
        int			 normal_prio;
        unsigned int		 rt_priority;
    }

1. prio是调度时候使用的优先级属性

2. static_prio则是用户设置nice度的时候, 根据nice转成内核优先级的数字

3. normal_prio和rt_priority从名字上就是一般性任务和实时任务的优先级, normal_prio则是和static_prio相同, 此时prio = normal_prio = static_prio

4. 实时任务的话是通过rt_priority计算的, 此时prio = func(rt_priority)

用户可以使用nice命令去提升某个进程的优先级(用户模式下也称为nice度), 用户能操作的优先级是-20-+19, 这些任务都是普通任务.

而内核中的优先级则是0-139这140个, 其中前100个属于实时任务(real time), 而100-139则是对应用户的-20-+19, 内核会转换的.

这140个数字, 越小表示优先级越高


上面四个属性在计算优先级的时候分别赋值, 当设置nice度的时候, 设置的是static_prio, 然后再计算task的其他三个优先级

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3819

.. code-block:: c

    void set_user_nice(struct task_struct *p, long nice)
    {
        // 其他代码先省略

        // 把nice度转成内核那种优先级
        // static_prio则是保存的是用户设置的优先级
    	p->static_prio = NICE_TO_PRIO(nice);
    	set_load_weight(p, true);
    	old_prio = p->prio;
        // 会判断task的类型, 返回实际的优先级
        // 也就是设置prio这个属性
    	p->prio = effective_prio(p);
    	delta = p->prio - old_prio;

        // 其他代码先省略
    }
    EXPORT_SYMBOL(set_user_nice);

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L836

.. code-block:: c

    static int effective_prio(struct task_struct *p)
    {
    	p->normal_prio = normal_prio(p);
    	/*
    	 * If we are RT tasks or we were boosted to RT priority,
    	 * keep the priority unchanged. Otherwise, update priority
    	 * to the normal priority:
    	 */
    	if (!rt_prio(p->prio))
    		return p->normal_prio;
    	return p->prio;
    }


https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L816

.. code-block:: c

    static inline int normal_prio(struct task_struct *p)
    {
    	int prio;
    
    	if (task_has_dl_policy(p))
    		prio = MAX_DL_PRIO-1;
    	else if (task_has_rt_policy(p))
                // 实时任务的话, 优先级是通过rt_priority计算的
    		prio = MAX_RT_PRIO-1 - p->rt_priority;
    	else
    		prio = __normal_prio(p);
    	return prio;
    }

1. 其中dl_policy则是判断task中的policy属性是否是SCHED_DEADLINE, *policy == SCHED_DEADLINE*

2. task_has_rt_policy则是判断task的policy是否是rt调度策略, *policy == SCHED_FIFO || policy == SCHED_RR*

3. 最后一般任务的话, 其prio就是用户设置的static_prio, \_\_normal_prio的操作是*return p->static_prio;*


所以

1. 一般任务的prio, normal_prio, static_prio三者值相同, 其他两个属性是通过static_prio属性赋值过去的

2. 实时任务的话, 则是通过rt_priority计算

load weight
==============

task获取到多少的timeslice, 取决于优先级(调度策略), 但是具体到多少的timeslice, 或者说timeslice的大小, 取决于load weight.

下面是load weight的定义表, 比如-20这个load_weight值就很大很大, 88761.

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L6924

.. code-block:: c

    /*
     * Nice levels are multiplicative, with a gentle 10% change for every
     * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
     * nice 1, it will get ~10% less CPU time than another CPU-bound task
     * that remained on nice 0.
     *
     * The "10% effect" is relative and cumulative: from _any_ nice level,
     * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
     * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
     * If a task goes up by ~10% and another task goes down by ~10% then
     * the relative distance between them is ~25%.)
     */
    const int sched_prio_to_weight[40] = {
     /* -20 */     88761,     71755,     56483,     46273,     36291,
     /* -15 */     29154,     23254,     18705,     14949,     11916,
     /* -10 */      9548,      7620,      6100,      4904,      3906,
     /*  -5 */      3121,      2501,      1991,      1586,      1277,
     /*   0 */      1024,       820,       655,       526,       423,
     /*   5 */       335,       272,       215,       172,       137,
     /*  10 */       110,        87,        70,        56,        45,
     /*  15 */        36,        29,        23,        18,        15,
    };

优先级的变化导致load weight变化, 然后load weight表示了占用cpu时间的百分比, 注释说没变化一级, 会有10%差距

算法如下, 参考 [1]_

a, b两个任务, 优先级都是0, 两人的load weight都是1024, 然后占cpu比率都是0.5 = 1024/(1024+1024)

然后a的优先级变为-1, 其load weight变为1277, 然后a的cpu占比0.55 ≅ 1277/(1024+1277), 而b的cpu占比0.45 ≅ 1024/(1024+1277), a, b差了10%

其中, 空闲类型的任务, 其load weight被设置成很小, 内核中定义是3

.. code-block:: c

    #define WEIGHT_IDLEPRIO    3

设置load weight

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L737
    static void set_load_weight(struct task_struct *p, bool update_load)
    {
    	int prio = p->static_prio - MAX_RT_PRIO;
    	struct load_weight *load = &p->se.load;
    
    	/*
    	 * SCHED_IDLE tasks get minimal weight:
    	 */
    	if (idle_policy(p->policy)) {
                // 如果是空闲任务, 则设置load weight为空闲
    		load->weight = scale_load(WEIGHT_IDLEPRIO);
    		load->inv_weight = WMULT_IDLEPRIO;
    		return;
    	}
    
    	/*
    	 * SCHED_OTHER tasks have to update their load when changing their
    	 * weight
    	 */
    	if (update_load && p->sched_class == &fair_sched_class) {
                // 如果是普通任务, 调用reweight_task
    		reweight_task(p, prio);
    	} else {
    		load->weight = scale_load(sched_prio_to_weight[prio]);
    		load->inv_weight = sched_prio_to_wmult[prio];
    	}
    }

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L2814
    void reweight_task(struct task_struct *p, int prio)
    {
        // 拿到task中的sched_entity
    	struct sched_entity *se = &p->se;

        // cfs的runqueue相关的属性
    	struct cfs_rq *cfs_rq = cfs_rq_of(se);

        // 当前sched_entity的load值
    	struct load_weight *load = &se->load;

        // 根据新的prio, 通过查表去得到新的weight的值
    	unsigned long weight = scale_load(sched_prio_to_weight[prio]);
    
        // 这个函数是操作sched_entity的
    	reweight_entity(cfs_rq, se, weight, weight);
    	load->inv_weight = sched_prio_to_wmult[prio];
    }


调度类
==========


内核会根据task的调度策略(policy这个属性)去决定task的调度类, 然后调用调度类的指定函数, 不关心调度类的具体实现, 这就是解耦了嘛

/kernel/sched/文件夹是调度的源码, 其中:

1. core.c中定义了调度类必须实现的一般性接口

2. fair.c: 一般(normal)task的调度策略, 也就是CFS

3. rt.c: 实时(real time)任务的调度策略

4. idle.c: 空闲(idle)task的调度策略



当一个task处于运行状态的时候, 内核调用enqueue_task, 该函数的作用是把指定的task加入到cpu的runqueue里面(优先级插入?)

*Each CPU(core) in the system has its own runqueue, and any task can be included in at most one runqueue;*

*A process scheduler’s job is to pick one task from a queue and assign it to run on a respective CPU(core).*


