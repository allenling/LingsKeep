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

.. [15] https://www.ibm.com/developerworks/linux/library/l-completely-fair-scheduler/

.. [16] http://linuxperf.com/?p=42

.. [17] https://blog.csdn.net/gatieme/article/details/52067748

.. [18] https://blog.csdn.net/gatieme/article/details/52067898

.. [19] https://blog.csdn.net/wudongxu/article/details/8574753

.. [20] https://blog.csdn.net/wudongxu/article/details/8574753

.. [21] http://www.linuxinternals.org/blog/2016/03/20/tif-need-resched-why-is-it-needed/

.. [22] https://blog.csdn.net/gatieme/article/details/52068016

.. [23] http://bbs.chinaunix.net/thread-3628238-1-1.html

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

参考16是cfs中几个细节的讲解

参考17是cfs中更新vruntime的流程的讲解, 感觉讲得还挺清楚的

参考18是cfs中enqueue_task和dequeue_task的流程, 和参考17是同一个作者

参考19, 20是调度参数的配置的解释, 比如sched_min_granularity_ns等等

参考21是抢占调度时候, TIF_NEED_RESCHED标志位的作用

参考22, 23是schedule函数中, pick_next_task函数的流程分析, 写得挺清楚的

2.6.23至今(4.15)linux已经是CFS调度为主了

**关键点: 以epoll和clone系统调用流程为入口, 查看cfs流程, cfs的load balance先略过**

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
===============

KThread是内核态线程, 是内核创建的task结构.

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

*The O(1) scheduler was much more scalable and incorporated interactivity metrics with numerous heuristics to determine whether tasks were I/O-bound or processor-bound. But the O(1) scheduler became unwieldy in the kernel. The large mass of code needed to calculate heuristics was fundamentally difficult to manage and, for the purist, lacked algorithmic substance.*

  --- 参考15

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

1. O(1)的操作在于bitmap和链表的pop(0)和append操作 

2. O(1)是没有抢占的!!!因为它是找bitmap中第一个被置为1的优先级, 去运行该优先级下的runqueue

3. O(1)根据task的平均睡眠时间去判断task是否是io密集, 然后这个过程计算复杂且容易出错


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

*Preemptions are caused by timer interrupts. A timer interrupt is basically a clock tick inside the kernel, the clock ticks 1000 times a second;*

*When an interrupt happens, scheduler has to decide whether to grant CPU to some other process and, if so, to which one. The amount of time a process gets to run is called
 timeslice of a process.*
  

正常调度, 注意是正常调度, 而不是所有的让出cpu的行为, 发生是每一个钟周期执行的, 内核中时钟周期是1/1000秒(1ms), 其他主动让出cpu的行为, 比如sleep/select等操作主动让出cpu, 也需要调度器

去决定下一个任务是哪一个.

但是, 每个时间周期内核都会去判断是否需要切换当前的task. 如果不需要切换task, 那么当前task则会运行下去

task运行的时间称为时间片段, timeslice. 如果task一直运行直到时钟中断, 那么task就完全利用了它的timeslice, 否则不能完全利用timeslice.

*A scheduling policy in the system has to balance between the two types of processes, and to make sure that every task gets enough execution resources, with no visible effect on the performance of
other jobs*

task分类型, 分为cpu密集和io密集, 显然io密集类型的task不是总能完全利用timeslice, 因为它会主动去等待io有发生, 而cpu密集型则总是完全利用. 

一个task不是严格区分类型的, 有可能某个时候是io密集, 某个时候是cpu密集. 调度器的责任则是平衡两种类型的task, 保证每一个task都能有足够的时间片段去运行, 确保cpu的最大利用率.

*To maximize CPU utilization and to guarantee fast response times, Linux tends to provide non-interactive processes with longer “uninterrupted” slices in a row, but to run them less
frequently. I/O bound tasks, in turn, possess the processor more often, but for shorter periods of time.*

为了最大化cpu的利用率, 同时保证task能快速响应, linux是让cpu密集型task运行时间更长, 但是频率(运行次数)不高, 而io类型的task则是运行时间很短, 但是运行次数很多.

CFS中的vruntime
==================

CFS中用红黑树存储task, 红黑树的key是task(sched_entity)中的vruntime属性的值. CFS会从红黑树中拿到下一个task, 而下一个task的是红黑树中的最左叶节点(left_most)

而CFS中会把最左叶节点给缓存起来的, 也就是查找的时候直接访问而不是要经过一个log(n)的查找过程.

vruntime的是这样子的, 每当从红黑树拿到下一个task去运行, 那么该task的vruntime就变大, 也就是其被放入到右子节点中, 然后剩下的vruntime比较小的task

就有机会运行了. 这样保证了某个task一定会被运行, 比如a, b两个task, a的runtime是10, b的是30, 然后a运行, 假设a的vruntime每次加5, 那么a运行了

6次之后, b就会被选中运行.

优先级高的task, vruntime的增加会比较慢, 而优先级低的task, 其vruntime会增加得比较快, 保证优先级高的运行时间更多. 上面的a和b两个task, a优先级高, 所以其vruntime

增加得比较慢, 一次加5. 所以a会比b运行次数(和时间)都会比b多.

vruntime增加的值则是公共task自身的优先级(也就是权重)计算出来的.

这里的vruntime是虚拟的运行时间, 在cfs中, 还保存了实际总运行的cpu时间, sum_exec_runtime, 所以两者是不同的. vruntime则是用来选择下一个task的, 而sum_exec_runtime

则是真实的已经运行过的cpu时间, 然后sum_exec_runtime和prev_sum_exec_runtime的差值得出运行的时间.

下面出自参考 [1]_

*when a task is executing, its virtual run time increases, so it moves to the right in the red-black tree;*

当一个task运行的时候, 其vruntime增加, 所以它被移动高右节点中

*virtual clock ticks more slowly for more important processes (those, having higher priorities), so they also move slower to the right in the rbtree and their chance to be scheduled again soon is
bigger than lower priority tasks’, for which the virtual clock ticks faster*

优先级高的task, 其vruntime增加得比较慢, 而优先级低的增加得快


所有cfs的整体结构就是:

1. 一颗全局红黑树

2. 每次从红黑树拿最左子节点, 该节点就是当前需要运行的task

3. 分配该task到cpu的runqueue


----

下面是代码
===============

会从下面几个流程去看cfs的调度源码:

1. 创建一个线程之后, 如果唤醒该新的线程

2. epoll陷入sleep的时候, 如何调度

3. epoll被唤醒之后, 如何调度

4. 定时的抢占流程

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
   SCHED_BATCH是说该任务会一直运行比较久, 就是适合那种cpu密集的任务了


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

这140个数字:

1. 实时任务的优先级比用户任务的优先级高, 也就是0-99比100-139优先级高

2. 在0-99中, 数字越大, 优先级比较高, 比如80比90的优先级高

3. 用户任务中, 也就是100-139中, 数字越小优先级越高, 也就是120比130的优先级高

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


而用户调用nice命令修改task的nice度的时候, 会去重新设置task的load weight的


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3783
    void set_user_nice(struct task_struct *p, long nice)
    {
        // 省略代码
    
        p->static_prio = NICE_TO_PRIO(nice);
        set_load_weight(p, true);
    
        // 省略代码
    
    }



调度类
==========


内核会根据task的调度策略(policy这个属性)去决定task的调度类, 然后调用调度类的指定函数, 不关心调度类的具体实现, 这就是解耦了嘛

/kernel/sched/文件夹是调度的源码, 其中:

1. core.c中定义了调度类必须实现的一般性接口

2. fair.c: 一般(normal)task的调度策略, 也就是CFS

3. rt.c: 实时(real time)任务的调度策略

4. idle.c: 空闲(idle)task的调度策略

下面主要是cfs的代码流程


调用路径
====================

从具体调用去看调度的流程, 下面是一些调用路径的总结


1. clone(_do_fork)中的调用:

.. code-block:: python

    '''
    _do_fork -> copy_process     -> sched_fork         -> task_fork(task_fork_fair)      -> update_curr (更新cfs_rq->curr的vruntime和sum_exec_runtime) -> update_min_vruntime (更新cfs_rq->min_vruntime)

                                                                                         -> place_entity (cfs一些补偿操作)

             -> wake_up_new_task -> activate_task      -> enqueue_task                   -> enqueue_task_fair (cfs)

                                 -> check_preempt_curr -> check_preempt_wakeup (cfs)
    '''

    所以流程上就是, 先调用copy_process, 对新建的task进行调度的初始化, 然后如果配置了子task要优先于父task运行, 则标识父task为被抢占状态

    然后经过copy_process之后, 父task被抢占之后选择的task不一定是新建的task, 而check_preempt_wakeup则是把当前task设置为cfs_rq->next或者cfs_rq->last

    这样父task被抢占走的时候, 选择的下一个task就很有可能是当前新建的task了

    此时传入给check_preempt_wakeup的标志位(wake_flag)是WF_FORK


2. epoll的唤醒中, 先把把current加入到waitqueue中之后, 初始化默认的回调函数, 就是默认的唤醒函数default_wake_function, 该函数调用try_to_wake_up

.. code-block:: python

    '''
    try_to_wake_up -> ttwu_queue -> ttwu_do_activate -> ttwu_active    -> activate_task(看上面)

                                                     -> ttwu_do_wakeup -> check_preempt_curr(看上面, 但是传入的参数不一样)
    
    
    '''

    try_to_wake_up给check_preempt_curr(check_preempt_wakeup)传入的唤醒标志位(wake_flag)是WF_SYNC


3. epoll中休眠等待事件发生, schedule_hrtimeout_range调用schedule函数去休眠放弃cpu的, 也就是强制做一次抢占操作, 移除curr, 强行放弃cpu

.. code-block:: python

    '''
    
    schedule -> __schedule -> deactivate_task -> dequeue_task               -> dequeue_task_fair (cfs)

                           -> pick_next_task  -> pick_next_task_fair (cfs)

                           -> context_switch (if prev != next)
    
    
    '''

4. schedule中pick_next_task流程

    .. code-block:: python
    
    pick_next_task -> pick_next_task_fair -> pick_next_entity
    
                                          -> put_prev_entity
    
                                          -> set_next_entity
    
    
    '''




5. enqueue的流程:

.. code-block:: python

    '''
    
    enqueue_task_fair -> enqueue_entity -> update_curr
                                        
                                        -> place_entity
    
                                        -> __enqueue_entity
    
    
    '''


6. check_preempt_curr流程, 其中还会配置去设置cfs_rq->next/cfs_rq->last, next和last是用来做抢占的时候, 和leftmost比较,  选择更合适的task来运行

.. code-block:: python

    '''
    
    check_preempt_curr -> check_preempt_wakeup (cfs) -> update_curr
                     
                                                     -> resched_curr(rq)
    
    
    '''


7. 时钟周期中关于调度的流程

.. code-block:: python

    '''
    
    scheduler_tick -> task_tick -> task_tick_fair -> entity_tick -> check_preempt_tick -> resched_curr
    
    '''

    主要是判断rq->curr是否需要被抢占走, 其他还包括load balance

* 其中check_preempt_tick和check_preempt_curr都会调用resched_curr, 但是条件是有区别的
  
  check_preempt_tick  : **计算req->curr的时间片是否使用完了, 使用完了则调用resched_curr去设置被抢占状态, 相关的计算属性是sum_exec_runtime/prev_sum_exec_runtime**

  check_preempt_wakeup: **判断传入的task的vruntime是否小于rq->curr, 如果小于, 则证明被唤醒的传入的task很渴望运行, 所以调用resched_curr设置rq->curr被抢占状态**

* update_curr都是更新cfs_rq->curr的vruntime和sum_exec_runtime, 以及cfs_rq->min_vruntime的值
  
  下次时钟周期去调用check_preempt_tick通过sum_exec_runtime和prev_sum_exec_runtime时间的差值
  
  去判断是cfs_rq->curr是否已经用完了 **计划分配的(理想的, ideal)运行时间**, 如果是, 则调用resched_curr设置cfs_rq->curr需要被抢占掉

* **值得注意的是, 上面的流程, 最终都是调用到resched_curr, 而resched_curr只是把rq->curr这个task设置上被抢占状态(TIF_NEED_RESCHED), 真正的去做一次抢占是在schedule(__schedule)函数**

  也就是谁调用schedule, 就是执行了一次强制抢占

* 进行一次抢占的时候是否一定是最左叶节点? 不一定, 会拿leftmost和cfs_rq->next, cfs_rq->last三者比较, 选一个合适的.

  这里配合check_preempt_wakeup和schedule一起看


clone
==========

在创建线程中, 调用了系统的clone系统调用, 其中会对新的task进行初始化, 然后再启动该新的task.

clone调用中, 调用\_do_fork函数, 其中:

1. 调用的copy_process初始化新的task结构

2. 调用wake_up_new_task启动新的task结构 

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L1534
    // 下面省略了很多很多代码
    long _do_fork(unsigned long clone_flags,
    	      unsigned long stack_start,
    	      unsigned long stack_size,
    	      int __user *parent_tidptr,
    	      int __user *child_tidptr,
    	      unsigned long tls)
    {
        p = copy_process(clone_flags, stack_start, stack_size, child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    
        if (!IS_ERR(p)) {
            wake_up_new_task(p);
        }
    
    }

sched_fork
===============

copy_process的中关于调度的处理是调用sched_fork函数, 在sched_fork函数中, 初始化vruntime等参数


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L2340
    int sched_fork(unsigned long clone_flags, struct task_struct *p)
    {
    	unsigned long flags;
    	int cpu = get_cpu();
    
        // 这里是初始化属性的地方!!!!!!!!!!!!!
    	__sched_fork(clone_flags, p);

        // 设置p->state属性, TASK_NEW = 0x0800
    	p->state = TASK_NEW;
    
    	/*
    	 * Make sure we do not leak PI boosting priority to the child.
    	 */
        // 注意这里!!!这里中把新的task结构的prio结构的值赋值为当前task的normal_prio的属性值
    	p->prio = current->normal_prio;
    
    	/*
    	 * Revert to default priority/policy on fork if requested.
    	 */
        // 这个if没看懂, 不过看到unlikely的编译标志, 也就是这个if很少会用到
        // 所以略过吧
        // 并且从注释可以出, sched_reset_on_fork标志位是说子task不继承父task的调度参数
        // 从而需要在这里重新计算的过程, 这里会根据子task的调度策略去计算
    	if (unlikely(p->sched_reset_on_fork)) {
    		if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
    			p->policy = SCHED_NORMAL;
    			p->static_prio = NICE_TO_PRIO(0);
    			p->rt_priority = 0;
    		} else if (PRIO_TO_NICE(p->static_prio) < 0)
    			p->static_prio = NICE_TO_PRIO(0);
    
    		p->prio = p->normal_prio = __normal_prio(p);
    		set_load_weight(p, false);
    
    		/*
    		 * We don't need the reset flag anymore after the fork. It has
    		 * fulfilled its duty:
    		 */
    		p->sched_reset_on_fork = 0;
    	}
    
        // 设置sched_class的地方, 一般被设置成fair_sched_class
    	if (dl_prio(p->prio)) {
    		put_cpu();
    		return -EAGAIN;
    	} else if (rt_prio(p->prio)) {
    		p->sched_class = &rt_sched_class;
    	} else {
    		p->sched_class = &fair_sched_class;
    	}
    
    	init_entity_runnable_average(&p->se);
    
    	/*
    	 * The child is not yet in the pid-hash so no cgroup attach races,
    	 * and the cgroup is pinned to this child due to cgroup_fork()
    	 * is ran before sched_fork().
    	 *
    	 * Silence PROVE_RCU.
    	 */
    	raw_spin_lock_irqsave(&p->pi_lock, flags);
    	/*
    	 * We're setting the CPU for the first time, we don't migrate,
    	 * so use __set_task_cpu().
    	 */
        // 设置cpu
    	__set_task_cpu(p, cpu);

        // 调用fair_sched_class中的task_fork
        // 这是为了进一步设置task的vruntime
    	if (p->sched_class->task_fork)
    		p->sched_class->task_fork(p);
    	raw_spin_unlock_irqrestore(&p->pi_lock, flags);
    
    #ifdef CONFIG_SCHED_INFO
    	if (likely(sched_info_on()))
    		memset(&p->sched_info, 0, sizeof(p->sched_info));
    #endif
    #if defined(CONFIG_SMP)
    	p->on_cpu = 0;
    #endif
    	init_task_preempt_count(p);
    #ifdef CONFIG_SMP
    	plist_node_init(&p->pushable_tasks, MAX_PRIO);
    	RB_CLEAR_NODE(&p->pushable_dl_tasks);
    #endif
    
    	put_cpu();
    	return 0;
    }

__sched_fork
===============

这个函数是初始化(设置0)task中的调度属性的地方

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L2166

.. code-block:: c
   
    /*
     * Perform scheduler related setup for a newly forked process p.
     * p is forked by current.
     *
     * __sched_fork() is basic setup used by init_idle() too:
     */
    static void __sched_fork(unsigned long clone_flags, struct task_struct *p)
    {
        // 初始化各种属性为0, 注意看vruntime和sum_exec_runtime, 还有prev_sum_exec_runtime都被设置为0
    	p->on_rq			= 0;
    	p->se.on_rq			= 0;
    	p->se.exec_start		= 0;
    	p->se.sum_exec_runtime		= 0;
    	p->se.prev_sum_exec_runtime	= 0;
    	p->se.nr_migrations		= 0;
    	p->se.vruntime			= 0;
    	INIT_LIST_HEAD(&p->se.group_node);

        // 下面的代码先省略
    
    }


**prev_sum_exec_runtime和sum_exec_runtime在check_preempt_tick中会被用来计算时间片是否用完, 往下看**


fair_sched_class->task_fork
==============================

sched_fork中, 最后调用fair_sched_class中的task_fork函数

在fair.c中, 该函数被定义为task_fork_fair

主要流程是:

1. update_curr : 更新cfs_rq->curr->vruntime, cfs_rq->min_vruntime

2. place_entity: 基于cfs_rq->min_runtime, 去设置(补偿)新建task的vruntime

3. 如果设置了子task必须比父task先运行(sysctl_sched_child_runs_first标志位),　并且父task的vruntime小于子task的vruntime

   则交换两个task的vruntime达到子task优先运行的目的

4. 唤醒的task经过补偿之后, vruntime很有可能比curr的小, 有很大概率上会把curr给抢占掉, 具体请看参考 [16]_

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9442

.. code-block:: c

    /*
     * called on fork with the child task as argument from the parent's context
     *  - child not yet on the tasklist
     *  - preemption disabled
     */
    static void task_fork_fair(struct task_struct *p)
    {
    	struct cfs_rq *cfs_rq;
    	struct sched_entity *se = &p->se, *curr;
    	struct rq *rq = this_rq();
    	struct rq_flags rf;
    
    	rq_lock(rq, &rf);
    	update_rq_clock(rq);
    
    	cfs_rq = task_cfs_rq(current);
    	curr = cfs_rq->curr;
    	if (curr) {
                // 这里调用update_curr去更新cfs中当前task的vruntime
    		update_curr(cfs_rq);
                // 这里!!!!!se的vruntime初始化为curr被更新之后的vruntime
    		se->vruntime = curr->vruntime;
    	}

        // 这里!!!上一个if代码里面, se被初始化为curr的vruntime值之后
        // 这个函数是对task的vruntime进行一些补偿
    	place_entity(cfs_rq, se, 1);
    
        // 这个判断是说如果配置了子线程在父亲现在之前运行的话
        // 确保子线程的vruntime大于父线程的vruntime, 也就是交换操作
        // entity_before则是比较第一个se的vruntime是否小于第二个se的vruntime
    	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
    		/*
    		 * Upon rescheduling, sched_class::put_prev_task() will place
    		 * 'current' within the tree based on its new key value.
    		 */
    		swap(curr->vruntime, se->vruntime);
                // 然后设置rq->curr为被抢占状态, 那么下一次检查是否需要被抢占的时候
                // rq->curr则会被抢占走的
    		resched_curr(rq);
    	}
    
    	se->vruntime -= cfs_rq->min_vruntime;
    	rq_unlock(rq, &rf);
    }

主要函数是:

1. update_curr, 以及update_curr中调用的update_min_vruntime

2. place_entity

update_curr
===============

更新cfs_rq->curr的vruntime属性和cfs_rq->min_vrumtime

主要参考 [17]_

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L819

.. code-block:: c

    /*
     * Update the current task's runtime statistics.
     */
    static void update_curr(struct cfs_rq *cfs_rq)
    {
        // 当前cfs中的当前task
    	struct sched_entity *curr = cfs_rq->curr;
        // 拿到实际时钟时间
    	u64 now = rq_clock_task(rq_of(cfs_rq));
    	u64 delta_exec;
    
    	if (unlikely(!curr))
    		return;
    
        // 这个delta就是上一次执行和当前时间的差值
    	delta_exec = now - curr->exec_start;
    	if (unlikely((s64)delta_exec <= 0))
    		return;
    
        // 更新开始执行的时间
    	curr->exec_start = now;
    
    	schedstat_set(curr->statistics.exec_max,
    		      max(delta_exec, curr->statistics.exec_max));
    
        // 增加sum_exec_runtime
    	curr->sum_exec_runtime += delta_exec;

    	schedstat_add(cfs_rq->exec_clock, delta_exec);
    
        // 增加vruntime
    	curr->vruntime += calc_delta_fair(delta_exec, curr);

        // 更新cfs_rq的min_vruntime
    	update_min_vruntime(cfs_rq);
    
    	if (entity_is_task(curr)) {
    		struct task_struct *curtask = task_of(curr);
    
    		trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
    		cgroup_account_cputime(curtask, delta_exec);
    		account_group_exec_runtime(curtask, delta_exec);
    	}
    
    	account_cfs_rq_runtime(cfs_rq, delta_exec);
    }


calc_delta_fair的代码流程是:

1. 如果curr.nice != NICE_0_LOAD, 则curr−>vruntime += delta_exec * (NICE_0_LOAD/curr−>se−>load.weight)

2. 如果curr.nice == NICE_0_LOAD, 则curr−>vruntime+=delta

也就是如果当前task的优先级是默认的0, 也就是120(0), 那么task的vruntime的增量则是delta值, 否则是delta乘以其优先级和默认优先级之间load weight的比例

所以, 优先级越高, load weight越大, 则delta越小, 则vruntime的变大得越慢.


update_min_vruntime
=====================

主要流程是, 比对cfs_rq->curr->vruntime和leftmost(se)-vruntime之间的最小值为m, 然后min_vruntime = max(min_vruntime, m)

update_min_vruntime, 这个函数是更新cfs_rq中, 最小的vruntime的, 之所以还需要一个cfs_rq的最小vruntime, 是因为插入红黑树的时候, 限制最小的vruntime值至少

大于该值. 比如新建一个task, 设置其vruntime=0(在copy_process中), 那么它在相当长的时间内都会保持抢占CPU的优势, 这样就不好, 所以需要min_vruntime去限制

最小大小(参考 [16]_)

主要参考 [16]_

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L515

.. code-block:: c

    static void update_min_vruntime(struct cfs_rq *cfs_rq)
    {
    	struct sched_entity *curr = cfs_rq->curr;
        // 拿到缓存的最左叶节点
    	struct rb_node *leftmost = rb_first_cached(&cfs_rq->tasks_timeline);
    
        // 当前min_vruntime的值
    	u64 vruntime = cfs_rq->min_vruntime;
    
    	if (curr) {
    	    if (curr->on_rq)
                vruntime = curr->vruntime;
    	    else
    	        curr = NULL;
    	}
    
    	if (leftmost) { /* non-empty tree */
    		struct sched_entity *se;
    		se = rb_entry(leftmost, struct sched_entity, run_node);
    
    		if (!curr)
    		    vruntime = se->vruntime;
    		else
    		    vruntime = min_vruntime(vruntime, se->vruntime);
    	}
    
    	/* ensure we never gain time by being placed backwards. */
    	cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
    #ifndef CONFIG_64BIT
    	smp_wmb();
    	cfs_rq->min_vruntime_copy = cfs_rq->min_vruntime;
    #endif
    }

1. 如果curr和se都存在,     那么min_vruntime = max(min_vruntime, min(curr->vruntime, se->vruntime))

2. 如果curr不存在而se存在, 那么min_vruntime = max(min_vruntime, se->vruntime)

3. 如果curr存在而se不存在, 那么min_vruntime = max(min_vruntime, curr->vruntime)

4. 如果curr和se都不存在,   那么min_vruntime = max(min_vruntime, min_vruntime)


place_entity
===============

task_fork_fair调用update_cur之后, 会对传入的task, 也就是新建的task, 中其vruntime进行补偿

这个函数不仅对新建task补偿, 在被唤醒的时候的task也有补偿

补偿的基础值是min_vruntime

更多参考 [16]_

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L3921

.. code-block:: c

    static void
    place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
    {
        // 这里是以min_vruntime作为基础
    	u64 vruntime = cfs_rq->min_vruntime;
    
    	/*
    	 * The 'current' period is already promised to the current tasks,
    	 * however the extra weight of the new task will slow them down a
    	 * little, place the new task so that it fits in the slot that
    	 * stays open at the end.
    	 */
        // initial表示新建的task
        // 并且设置了START_DEBIT标志位
    	if (initial && sched_feat(START_DEBIT))
    		vruntime += sched_vslice(cfs_rq, se);
    
    	/* sleeps up to a single latency don't count. */
    	if (!initial) {
    		unsigned long thresh = sysctl_sched_latency; /* 一个调度周期 */
    
    		/*
    		 * Halve their sleep time's effect, to allow
    		 * for a gentler effect of sleepers:
    		 */
                // 如果设置了GENTLE_FAIR_SLEEPERS标志
    		if (sched_feat(GENTLE_FAIR_SLEEPERS))
    			thresh >>= 1; /* 补偿减为调度周期的一半, 右移一位就是除以2 */
    
    		vruntime -= thresh;
    	}
    
    	/* ensure we never gain time by being placed backwards. */
        // 补偿的vruntime和自己的vruntime, 取一个最大值
    	se->vruntime = max_vruntime(se->vruntime, vruntime);
    }

关于sched_features:

  *sched_features是控制调度器特性的开关，每个bit表示调度器的一个特性。在sched_features.h文件中记录了全部的特性.
  START_DEBIT是其中之一，如果打开这个特性，表示给新进程的vruntime初始值要设置得比默认值更大一些，这样会推迟它的运行时间，以防进程通过不停的fork来获得cpu时间*

  --- 参考16

新建task的补偿:

1. 补偿的基础, 也就是初始值是min_vruntime, 记得在sched_fork中, 把新建的task的vruntime初始化为0了

2. 如果是新建task, 并且规定新建的task第一次启动需要延迟, 则调用sched_vslice计算补偿, vruntime += sched_vslice

被唤醒task的补偿:

1. 默认是一个调度周期, thresh=sysctl_sched_latency

2. 如果设置了GENTLE_FAIR_SLEEPERS标志位, 那么补偿的值减少一半, thresh >>= 1

最后, 取补偿vruntime和se自己的vruntime的最大值

之所以是用min_vruntime作为基础来补偿, 是因为这样被唤醒的task的vruntime就接近于min_vruntime, 这样很快被调用, 但又不至于太小
而占据了很长的cpu时间(参考 [18]_)


update_curr/place_entity中的补偿
==================================

先来总结一下update_curr/place_entity中涉及到的补偿的流程, 其中place_entity主要是新建的, 先忽略被唤醒的请看:

.. code-block:: python

    '''
    
    update_curr  -> delta_exec = now - curr->exec_start
    
                 -> curr->vruntime += calc_delta_fair(delta_exec, curr) -> __calc_delta(delta, NICE_0_LOAD, &se->load) (如果task的优先级不是0)
    
    
    
    
    place_entity -> vruntime = cfs_rq->min_vruntime
    
                 -> vruntime += sched_vslice(cfs_rq, se) -> calc_delta_fair(sched_slice(cfs_rq, se), se)
    
    '''

我们看到, 两者都是调用 **calc_delta_fair** 去计算补偿的值, calc_delta_fair根据传入的delta和se, 计算公式是:

1. 如果se.nice != NICE_0_LOAD, 则new_delta = delta_exec * (NICE_0_LOAD/curr−>se−>load.weight)

2. 如果se.nice == NICE_0_LOAD, 则new_delta = delta

**然后两者传参是有区别的**:

1. update_curr的时候, 传入的delta是curr两次开始执行的时间的差值, 也就是curr->exec_start和now的差值

   比如curr上次执行的时间, curr->exec=100, 当前时间now=105, 那么delta = 105 - 100, 然后curr->exec_start = 105


2. 而place_entity中针对新建task的补偿中, 传入的delta则是通过sched_slice计算出来的, sched_slice的是

   根据cfs_rq中的运行的task的数量计算出来的


sched_slice
================

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L677

拿到一个基准的slice, 然后slice乘以se->load在整个cfs_rq中占据的比例, slice = slice * (se->load / cfs_rq->load)

.. code-block:: c

    static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);
    
    	for_each_sched_entity(se) {
    	    struct load_weight *load;
    	    struct load_weight lw;
    
    	    cfs_rq = cfs_rq_of(se);
    	    load = &cfs_rq->load;
    
    	    if (unlikely(!se->on_rq)) {
    	    	lw = cfs_rq->load;
    
    	    	update_load_add(&lw, se->load.weight);
    	    	load = &lw;
    	    }
    	    slice = __calc_delta(slice, se->load.weight, load);
    	}
    	return slice;
    }


*__sched_period* 先通过cfs_rq上正在运行的task的总数, 计算调度延迟, 这个调度延迟是算出来的, 会变化

.. code-blocl:: c

    static u64 __sched_period(unsigned long nr_running)
    {
    	if (unlikely(nr_running > sched_nr_latency))
    		return nr_running * sysctl_sched_min_granularity;
    	else
    		return sysctl_sched_latency;
    }

sysctl_sched_min_granularity: task的最小运行时间

关于sysctl_sched_latency, sysctl_sched_latency, sysctl_sched_min_granularity等这些参数, 参考 [19]_ , 参考 [20]_, 参考[16]_

  *假设有两个进程，它们的vruntime初值都是一样的，第一个进程只要一运行，它的vruntime马上就比第二个进程更大了，那么它的CPU会立即被第二个进程抢占吗？答案是这样的：为了避免过于短暂的进程切换造成太大的消耗，CFS设定了进程占用CPU的最小时间值，sched_min_granularity_ns，正在CPU上运行的进程如果不足这个时间是不可以被调离CPU的。*
  
  -- 参考16
  
  *ched_min_granularity_ns发挥作用的另一个场景是，本文开门见山就讲过，CFS把调度周期sched_latency按照进程的数量平分，给每个进程平均分配CPU时间片（当然要按照nice值加权，为简化起见不再强调），但是如果进程数量太多的话，就会造成CPU时间片太小，如果小于sched_min_granularity_ns的话就以sched_min_granularity_ns为准；而调度周期也随之不再遵守sched_latency_ns，而是以 (sched_min_granularity_ns * 进程数量) 的乘积为准*
  
  -- 参考16

其中, sched_nr_latency是配置好的, 固定的, 默认值是8, 其值是sysctl_sched_latency除以sysctl_sched_min_granularity

也就是无论sysctl_sched_latency和sysctl_sched_min_granularity怎么变(是会变的), 两者相除一定是8(这个存疑~~~但是看代码注释是这样说的)

https://elixir.bootlin.com/linux/latest/source/kernel/sched/fair.c#L55

.. code-block:: c

    // 默认6ms
    unsigned int sysctl_sched_latency			= 6000000ULL;
    
    // 默认是0.75ms
    unsigned int sysctl_sched_min_granularity		= 750000ULL;
    
    /*
     * This value is kept at sysctl_sched_latency/sysctl_sched_min_granularity
     */
    static unsigned int sched_nr_latency = 8;


所以, __sched_period的流程就是

1. 如果正在运行的进程数大于sched_nr_latency, 那么调度周期就是总个数 * 最小运行时间

2. 否则, 就是一个调度周期sysctl_sched_latency

我们得到了一个基准的调度周期值, 然后接下来调用__calc_delta去根据se的load_weight去更新

也就是说, 一个基准的slice, 然后有slice = __calc_delta(slice, se->load.weight, load);

而__calc_delta的公式是: 第一个参数 * (第二个参数/第三个参数), 根据上面的传参可知, slice最终的值slice = slice * (se->load / cfs_rq-load)

**也就是说, se的slice是自己的load占据整个cfs_rq->load的比例** 

**关于里面的for循环, 是和cfs group scheduling有关, 这里先讨论非组调度的情况, 所以for循环其实只循环了给的se**

而关于实际抢占是否发生, 还和sched_min_granularity_ns等参数有关(参考 [16]_), 具体继续看后面


place_entity补偿最终计算
==========================

先调用通过sched_slice得到delta, 然后place_entity再次调用calc_delta_fair去计算最终的vruntime

1. new_for_task->vruntime = min_vruntime + sched_vslice(cfs_rq, se)
                         
                            min_vruntime + calc_delta_fair(sched_slice(cfs_rq, se), se)

2. delta = sched_slice(cfs_rq, se), delta = slice = base_slice * (se->load / cfs_rq->load)

2. calc_delta_fair(sched_slice(cfs_rq, se), se) = calc_delta_fair(slice, NICE_0_LOAD, se)

3. 所以, new_for_task->vruntime = min_vruntime + slice * (NICE_0_LOAD / se->load)

                                = min_vruntime + base_slice * (se->load / cfs_rq->load) * (NICE_0_LOAD / se->load)

place_entity交换父子task的vruntime
=======================================

在place_entity中, 计算了新创建的task的实际vruntime之后, 会根据是否配置了sysctl_sched_child_runs_first标志

去决定是否交换父子task之间的vruntime


如果配置了sysctl_sched_child_runs_first, 所以fork出来的子task的vruntime, 也就是经过 *min_vruntime +=sched_vslice* 计算之后

的值大于父task(curr)的vruntime, 说明父task还是会先于子task运行, 那么交换两者的vruntime, 然后调用resched_curr去标识

curr需要被抢占走


.. code-block:: c

    	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
    		/*
    		 * Upon rescheduling, sched_class::put_prev_task() will place
    		 * 'current' within the tree based on its new key value.
    		 */
    		swap(curr->vruntime, se->vruntime);
                // 然后设置rq->curr为被抢占状态, 那么下一次检查是否需要被抢占的时候
                // rq->curr则会被抢占走的
    		resched_curr(rq);
    	}


**注意的是**:

这里虽然调用resched_curr去标识curr应该被抢占走, 但是这里并没有把子task加入到cfs中

也就是说虽然curr被标识被抢占走, 但是下一个task不一定是当前这个新建的task, 所以需要做一些操作去提醒cfs下一个

运行的task是这个新建的task, 这就需要交给下面wake_up_new_task来操作了

wake_up_new_task
===================

这个函数是_do_fork中唤醒新task结构的地方

https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L2015

.. code-block:: c

    long _do_fork(unsigned long clone_flags,
    	      unsigned long stack_start,
    	      unsigned long stack_size,
    	      int __user *parent_tidptr,
    	      int __user *child_tidptr,
    	      unsigned long tls)
    {
    
        p = copy_process(clone_flags, stack_start, stack_size,
        			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
        if (!IS_ERR(p)) {
            wake_up_new_task(p);
        }
    }

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L2447


.. code-block:: c

    /*
     * wake_up_new_task - wake up a newly created task for the first time.
     *
     * This function will do some initial scheduler statistics housekeeping
     * that must be done for every newly created context, then puts the task
     * on the runqueue and wakes it.
     */
    void wake_up_new_task(struct task_struct *p)
    {
    	struct rq_flags rf;
    	struct rq *rq;
    
    	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
        // 把task的state赋值为TASK_RUNNING
    	p->state = TASK_RUNNING;
    #ifdef CONFIG_SMP
    	/*
    	 * Fork balancing, do it here and not earlier because:
    	 *  - cpus_allowed can change in the fork path
    	 *  - any previously selected CPU might disappear through hotplug
    	 *
    	 * Use __set_task_cpu() to avoid calling sched_class::migrate_task_rq,
    	 * as we're not fully set-up yet.
    	 */
        // SMP架构下, load balance可能会改变cpu
        // 注释上的原因是说: 1. cpus_allowed可能在fork的过程中会变化 2. 之前选择的cpu可能不见了, 比如被禁用了.
    	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
    #endif
    	rq = __task_rq_lock(p, &rf);
    	update_rq_clock(rq);
    	post_init_entity_util_avg(&p->se);
    
        // 这个是唤醒的主要函数, 主要是调用enqueue去
        // 把task设置到cfs中的红黑树中
    	activate_task(rq, p, ENQUEUE_NOCLOCK);
        // 设置on_req为1
    	p->on_rq = TASK_ON_RQ_QUEUED;
    	trace_sched_wakeup_new(p);
    	check_preempt_curr(rq, p, WF_FORK);
    #ifdef CONFIG_SMP
        // cfs中并没有定义task_woken属性, 下面的代码过了
    	if (p->sched_class->task_woken) {
    		/*
    		 * Nothing relies on rq->lock after this, so its fine to
    		 * drop it.
    		 */
    		rq_unpin_lock(rq, &rf);
    		p->sched_class->task_woken(rq, p);
    		rq_repin_lock(rq, &rf);
    	}
    #endif
    	task_rq_unlock(rq, p, &rf);
    }


1. 设置task的状态为TASK_RUNNING, 然后如果在SMP架构下, 需要再次设置cpu(因为1. cpu_allowed可能有变化 2. 之前选择的cpu可能不可用了)

2. 调用activate_task函数去调用相关调度类的enqueue_task函数, 把task加入到cfs自己的红黑树中

3. 注意的是, **wake_up_new_task传给activate_task的flag不是ENQUEUE_WAKEUP, 而是ENQUEUE_NOCLOCK, 所以后面的操作不会调用place_entity去补偿task的vruntime**

4. 调用check_preempt_curr去做一次抢占操作, 传入的唤醒标志位是 **WF_FORK**, 表示这次唤醒是新建的task


activate_task/enqueue_task
==============================

activeate_task是直接调用enqueue_task, 而enqueue_task函数则是调用task自己的调度类的enqueue_task函数

.. code-block:: c

    void activate_task(struct rq *rq, struct task_struct *p, int flags)
    {
    	if (task_contributes_to_load(p))
    		rq->nr_uninterruptible--;
    
    	enqueue_task(rq, p, flags);
    }

    static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
    {
    	if (!(flags & ENQUEUE_NOCLOCK))
    		update_rq_clock(rq);
    
    	if (!(flags & ENQUEUE_RESTORE))
    		sched_info_queued(rq, p);
    
    	p->sched_class->enqueue_task(rq, p, flags);
    }

在cfs中, enqueue_task是enqueue_task_fair函数

enqueue_task_fair
================================

enqueue_task_fair的主要操作是把目标task给加入到cfs的红黑树中

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L5206

.. code-block:: c

    /*
     * The enqueue_task method is called before nr_running is
     * increased. Here we update the fair scheduling stats and
     * then put the task into the rbtree:
     */
    static void
    enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
    {
    	struct cfs_rq *cfs_rq;
    	struct sched_entity *se = &p->se;
    
    	/*
    	 * If in_iowait is set, the code below may not trigger any cpufreq
    	 * utilization updates, so do it here explicitly with the IOWAIT flag
    	 * passed.
    	 */
    	if (p->in_iowait)
    	    cpufreq_update_util(rq, SCHED_CPUFREQ_IOWAIT);
    
        
        // 这个循环是从传入的task开始
    	for_each_sched_entity(se) {
    	    if (se->on_rq)
    	    	break;
    	    cfs_rq = cfs_rq_of(se);
            // 这个函数是插入红黑树
    	    enqueue_entity(cfs_rq, se, flags);
    
    	    /*
    	     * end evaluation on encountering a throttled cfs_rq
    	     *
    	     * note: in the case of encountering a throttled cfs_rq we will
    	     * post the final h_nr_running increment below.
    	     */
    	    if (cfs_rq_throttled(cfs_rq))
    	    	break;
    	    cfs_rq->h_nr_running++;
    
    	    flags = ENQUEUE_WAKEUP;
    	}
    
    	for_each_sched_entity(se) {
    		cfs_rq = cfs_rq_of(se);
    		cfs_rq->h_nr_running++;
    
    		if (cfs_rq_throttled(cfs_rq))
    			break;
    
    		update_load_avg(cfs_rq, se, UPDATE_TG);
    		update_cfs_group(se);
    	}
    
    	if (!se)
    		add_nr_running(rq, 1);
    
    	hrtick_update(rq);
    }

**如果是新建的task, 那么在copy_process初始化的on_rq是0, 所以会走到第一个for循环的enqueue_entity函数**

关于第一个for循环

  *但是有个疑问是, 进程p所在的调度时提就一个, 为嘛要循环才能遍历啊?这是因为为了支持组调度.组调度下调度实体是有层次结构的, 我们将进程加入的时候, 同时要更新其父调度实体的调度信息, 而非组调度情况下, 就不需要调度实体的层次结构*

  --- 参考18

**至于第二个for循环干嘛的, 不清楚!**

enqueue_entity加入红黑树
==========================

参考 [18]_

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L4006

.. code-block:: c

    static void
    enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
    {
    	bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);

        // 判断下是, 传入的task和cfs_rq->curr当前否是同一个
    	bool curr = cfs_rq->curr == se;
    
    	/*
    	 * If we're the current task, we must renormalise before calling
    	 * update_curr().
    	 */
    	if (renorm && curr)
    	    se->vruntime += cfs_rq->min_vruntime;
    
        // 更新一下cfs_rq->curr->vruntime
    	update_curr(cfs_rq);
    
    	/*
    	 * Otherwise, renormalise after, such that we're placed at the current
    	 * moment in time, instead of some random moment in the past. Being
    	 * placed in the past could significantly boost this task to the
    	 * fairness detriment of existing tasks.
    	 */
    	if (renorm && !curr)
    	    se->vruntime += cfs_rq->min_vruntime;
    
    	/*
    	 * When enqueuing a sched_entity, we must:
    	 *   - Update loads to have both entity and cfs_rq synced with now.
    	 *   - Add its load to cfs_rq->runnable_avg
    	 *   - For group_entity, update its weight to reflect the new share of
    	 *     its group cfs_rq
    	 *   - Add its new weight to cfs_rq->load.weight
    	 */
        // 更新统计量
    	update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH);
    	update_cfs_group(se);
    	enqueue_runnable_load_avg(cfs_rq, se);
    	account_entity_enqueue(cfs_rq, se);
    
        // 这里, 如果是休眠而唤醒的进程, 调用place_entity去补偿
        // 显然, wake_up_new_task中传入的flag并不是ENQUEUE_WAKEUP
        // 所以不会走place_entity
    	if (flags & ENQUEUE_WAKEUP)
    	    place_entity(cfs_rq, se, 0);
    
    	check_schedstat_required();
    	update_stats_enqueue(cfs_rq, se, flags);
    	check_spread(cfs_rq, se);
        // 这里curr是一个真假值
        // 表示传入的task和cfs->curr是否一致, 也就是是否是同一个
    	if (!curr)
    	    __enqueue_entity(cfs_rq, se);
        // on_rq的属性设置为1
    	se->on_rq = 1;
    
    	if (cfs_rq->nr_running == 1) {
    	    list_add_leaf_cfs_rq(cfs_rq);
    	    check_enqueue_throttle(cfs_rq);
    	}
    }

1. 关于renorm的判断, 这里有可能是说task从另外一个cfs_rq(也可说是另外一个cpu)移动当前的cfs_rq(当前的cpu)中

   所以需要补偿, 参考 [16]_

2. 调用update_curr更新cfs_rq->curr的vruntime

3. 根据传入的flags中是否有ENQUEUE_WAKEUP标志去决定, 是否去调用place_entity去补偿vruntime

   显然, 在wake_up_new_task中传入的不是ENQUEUE_WAKEUP标志, 所以不会走vruntime

   **后面的epoll唤醒流程(default_wake_function -> try_to_wake_up)可以看到传入的flags中带有ENQUEUE_WAKEUP标志**

4. 更新其他统计量, 然后设置se->on_rq=1

5. 如果cfs_rq->curr和传入的task不是同一个, 则调用__enqueue_entity, 把传入的task加入到红黑树.

   __enqueue_entity的流程只是加入红黑树, **并且去判断是否是leftmost, 是的话设置新的leftmost节点**, 代码先省略吧

check_preempt_curr
======================

wake_up_new_task调用activate_task去调用enqueue_task, 把task加入到cfs的红黑树之后, 然后调用check_preempt_curr去做抢占操作

注意的是, 这里调用check_preempt_curr的时候, 传入的wake_flag是WF_FORK

.. code-block:: c

    void wake_up_new_task(struct task_struct *p)
    {
    
        activate_task(rq, p, ENQUEUE_NOCLOCK);
        
        check_preempt_curr(rq, p, WF_FORK);
    
    }


https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L880

.. code-block:: c

    void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags)
    {
    	const struct sched_class *class;
    
        // 这里判断task的调度类和rq的调度类是否一致
        // 然后我们简单点, 假设是一直并且是cfs
    	if (p->sched_class == rq->curr->sched_class) {
    		rq->curr->sched_class->check_preempt_curr(rq, p, flags);
    	} else {
    		for_each_class(class) {
    			if (class == rq->curr->sched_class)
    				break;
    			if (class == p->sched_class) {
    				resched_curr(rq);
    				break;
    			}
    		}
    	}
    
    	/*
    	 * A queue event has occurred, and we're going to schedule.  In
    	 * this case, we can save a useless back to back clock update.
    	 */
    	if (task_on_rq_queued(rq->curr) && test_tsk_need_resched(rq->curr))
    		rq_clock_skip_update(rq, true);
    }


如果task的调度类和rq->curr的调度类一致, 那么调用调度类的check_preempt_curr, 这里假设一直并且是cfs

则会调用到cfs中的check_preempt_wakeup, 该函数会判断是否需要去抢占

check_preempt_wakeup
=========================

主要是判断是否调用resched_curr, 为curr加上被抢占的标识

流程是:

1. 先判断是否需要set_next_buddy去设置cfs_rq->next, 无论是否调用set_next_buddy, 都走2

2. 判断当前task(curr)是否被加上了抢占标识(TIF_NEED_RESCHED), 如果已经被加上了, 则退出, 否则走3
   
3. 调用wakeup_preempt_entity去判断传入的task是否应该抢占掉curr, 如果应该抢占, 则调用resched_curr

4. 是否需要调用set_last_buddy设置cfs_rq->last

设置curr为需要被抢占状态.

需要注意一下的是cfs->next, cfs->last的设置, 相关的调度配置有, NEXT_BUDDY, LAST_BUDDY, WAKEUP_PREEMPTION


.. code-block:: c

    static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
    {
        // rq的当前task
    	struct task_struct *curr = rq->curr;
        // 获取对应的se
    	struct sched_entity *se = &curr->se, *pse = &p->se;
        // 获取cfs_rq
    	struct cfs_rq *cfs_rq = task_cfs_rq(curr);
    	int scale = cfs_rq->nr_running >= sched_nr_latency;

        // 注意一下next_buddy的配置
    	int next_buddy_marked = 0;
    
    	if (unlikely(se == pse))
    		return;
    
    	/*
    	 * This is possible from callers such as attach_tasks(), in which we
    	 * unconditionally check_prempt_curr() after an enqueue (which may have
    	 * lead to a throttle).  This both saves work and prevents false
    	 * next-buddy nomination below.
    	 */
    	if (unlikely(throttled_hierarchy(cfs_rq_of(pse))))
    	    return;
    
        // 如果开启了NEXT_BUDDY特性, 然后传入的wake_flags有WF_FORK
        // 设置cfs_rq->next = pse
    	if (sched_feat(NEXT_BUDDY) && scale && !(wake_flags & WF_FORK)) {
    	    set_next_buddy(pse);
    	    next_buddy_marked = 1;
    	}
    
    	/*
    	 * We can come here with TIF_NEED_RESCHED already set from new task
    	 * wake up path.
    	 *
    	 * Note: this also catches the edge-case of curr being in a throttled
    	 * group (e.g. via set_curr_task), since update_curr() (in the
    	 * enqueue of curr) will have resulted in resched being set.  This
    	 * prevents us from potentially nominating it as a false LAST_BUDDY
    	 * below.
    	 */
    	if (test_tsk_need_resched(curr))
    	    return;
    
    	/* Idle tasks are by definition preempted by non-idle tasks. */
    	if (unlikely(curr->policy == SCHED_IDLE) &&
    	    likely(p->policy != SCHED_IDLE))
    		goto preempt;
    
    	/*
    	 * Batch and idle tasks do not preempt non-idle tasks (their preemption
    	 * is driven by the tick):
    	 */
        // 如果没有开启WAKEUP_PREEMPTION特性, 那么
        // 唤醒的task是不能抢占当前task的, 也就是必须等待当前task把时间片消耗完
    	if (unlikely(p->policy != SCHED_NORMAL) || !sched_feat(WAKEUP_PREEMPTION))
    	    return;
    
    	find_matching_se(&se, &pse);
    	update_curr(cfs_rq_of(se));
    	BUG_ON(!pse);
        // 比对一下传入的task, 也就是唤醒的task, 和当前task那个优先级高(vruntime哪个小)
        // 如果传入的task优先级高, 那么需要调用resched_curr
        // 否则退出
    	if (wakeup_preempt_entity(se, pse) == 1) {
    	    /*
    	     * Bias pick_next to pick the sched entity that is
    	     * triggering this preemption.
    	     */
            // 如果开启了NEXT_BUDDY特性, 设置task为cfs_rq->next
    	    if (!next_buddy_marked)
    	    	set_next_buddy(pse);
    	    goto preempt;
    	}
    
    	return;
    
    preempt:
    	resched_curr(rq);
    	/*
    	 * Only set the backward buddy when the current task is still
    	 * on the rq. This can happen when a wakeup gets interleaved
    	 * with schedule on the ->pre_schedule() or idle_balance()
    	 * point, either of which can * drop the rq lock.
    	 *
    	 * Also, during early boot the idle thread is in the fair class,
    	 * for obvious reasons its a bad idea to schedule back to it.
    	 */
    	if (unlikely(!se->on_rq || curr == rq->idle))
    	    return;
    
        // 如果开启了LAST_BUDDY特性, 把task设置到cfs_rq->last上
    	if (sched_feat(LAST_BUDDY) && scale && entity_is_task(se))
    	    set_last_buddy(se);
    }


调度特性, 是值调度的一些配置, 在https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/features.h

其中和上面提到的调度特性有, 下面是默认配置

.. code-block:: c

    // 新建的task的vruntime应该有演出
    // 也就是place_entity中会对新建的task的vruntime进行sched_vslice补偿
    SCHED_FEAT(START_DEBIT, true)
    
    /*
     * Prefer to schedule the task we woke last (assuming it failed
     * wakeup-preemption), since its likely going to consume data we
     * touched, increases cache locality.
     */
    // 在选择下一个task的时候, 更倾向于cfs_rq->next
    SCHED_FEAT(NEXT_BUDDY, false)
    
    /*
     * Prefer to schedule the task that ran last (when we did
     * wake-preempt) as that likely will touch the same data, increases
     * cache locality.
     */
    // 在选择下一个task的时候, 更倾向于cfs_rq->last
    SCHED_FEAT(LAST_BUDDY, true)
    
    // 唤醒的task一定会抢占掉当前task
    SCHED_FEAT(WAKEUP_PREEMPTION, true)

当然, 也可以在/sys/kernel/debug/sched_features下看到内核所配置的特性

关于WAKEUP_PREEMPTION特性, 可以参考 [16]_, 关闭这个特性的话, 唤醒的task不会抢占掉正在运行的task了

**根据默认配置的特性, 可知, 一般会把传入的task设置到cfs_rq->last上**

关于cfs_rq->next, cfs_rq->last, 会关系到下一个task的选择.

也就是说, 下一个task的选择可能不一定是leftmost, 有些task更需要运行, 这些更需要运行的task会设置到cfs_rq->next, cfs_rq->last上

这样就比对lestmost, next, last(比对有个算法, 不是简单的比较), 选一个更合适的task. next和last也可以在选择的时候可以直接拿而不是查找

相当于缓存了最想运行的task

  *pick_next_entity的代码，它选择进程的规则较书上说的已经有了一些改进。原本cfs总是选择rb树最左边的进程，也就是虚拟时钟最落后的进程。现在又在这个规则之上加入了buddy这个概念*
  
  -- 参考23 

关于下一个task的选择, 看后面的schedule部分

**注意的地方:** 有两个set_next_buddy的地方

1. 第一个调用条件是: 设置了NEXT_BUDDY特性, 并且cfs_rq->nr_running大于sched_nr_latency, 并且传入的flag不包含WF_FORK

2. 第二个调用条件是: 如果传入的task确实应该被选择, 并且1中的判断没有通过, 则强制调用set_next_buddy


wakeup_preempt_entity
=========================

作用是判断传入的curr和se之间, se是否应该抢占掉curr, 抢占的条件是se要小于curr, 并且大于curr+gran

这个函数在schedule中也有使用

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L6488

.. code-block:: c

    /*
     * Should 'se' preempt 'curr'.
     *
     *             |s1
     *        |s2
     *   |s3
     *         g
     *      |<--->|c
     *
     *  w(c, s1) = -1
     *  w(c, s2) =  0
     *  w(c, s3) =  1
     *
     */
    static int
    wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se)
    {
        // curr->vruntime和se->vruntime的差值
    	s64 gran, vdiff = curr->vruntime - se->vruntime;
    
        // 如果curr->vruntime小于等于se->vruntime
        // 那么显然curr不能被抢占
    	if (vdiff <= 0)
    		return -1;
    
        // 计算一下curr可允许的抢占范围
    	gran = wakeup_gran(curr, se);
        // 如果se->vruntime小于curr->vrutime
        // 并且差值比curr的某一个范围还要大, 则se可以抢占掉curr
        // 也就是说, se->vruntime必须比curr小, 并且小得多(小于curr再减去一个值)
    	if (vdiff > gran)
    		return 1;
    
    	return 0;
    }


看注释, c是curr, g是c的抢占范围

1. 如果传入的task在s1的位置, 也就是vdiff <= 0, 则返回-1

2. 如果传入的task是s2的位置, 也就是vdiff > 0, curr->vruntime > se->vruntime,

   但是, vdiff < gran, 也就是s2 - curr < g, 所以返回0

3. 如果是s3的位置, 那么明显vdiff > 0, 并且vidff > gran, 此时se才运行抢占掉curr!!!!!!!!!!!!!!!!!

而这个gran的值则是基于最小调度周期(sysctl_sched_min_granularity)计算的

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L6453

.. code-block:: c

    static unsigned long
    wakeup_gran(struct sched_entity *curr, struct sched_entity *se)
    {
    	unsigned long gran = sysctl_sched_wakeup_granularity;
    
    	/*
    	 * Since its curr running now, convert the gran from real-time
    	 * to virtual-time in his units.
    	 *
    	 * By using 'se' instead of 'curr' we penalize light tasks, so
    	 * they get preempted easier. That is, if 'se' < 'curr' then
    	 * the resulting gran will be larger, therefore penalizing the
    	 * lighter, if otoh 'se' > 'curr' then the resulting gran will
    	 * be smaller, again penalizing the lighter task.
    	 *
    	 * This is especially important for buddies when the leftmost
    	 * task is higher priority than the buddy.
    	 */
    	return calc_delta_fair(gran, se);
    }


还记得calc_delta_fair函数么, 这个函数的第一个参数是delta, 然后判断

1. 如果se的load是NICE_0_LOAD, 那么返回delta

2. 否则, 返回delta * (NICE_0_LOAD / se->load)

所以, 如果se->vruntime < curr->vruntime, 并且se->vruntime < curr->vruntime - (sysctl_sched_wakeup_granularity * (NICE_0_LOAD / se->load))

则说明传入的task可以抢占掉curr, 然后调用resched_curr, 并且设置task为cfs_rq->next或csf_rq->last, 是得curr被抢占掉的时候能优先选择传入的task

resched_curr
=================

这个函数主要作用是把rq->curr这个task加上需要被抢占的标识(TIF_NEED_RESCHED), 这样某个地方(后面说)

判断当前的curr是被设置上了被抢占的标识, 则强行进行一次抢占操作

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L481

.. code-block:: c

    void resched_curr(struct rq *rq)
    {
    	struct task_struct *curr = rq->curr;
    	int cpu;
    
    	lockdep_assert_held(&rq->lock);
    
        // 如果curr已经被设置过抢占标识了
        // 退出
    	if (test_tsk_need_resched(curr))
    		return;
    
    	cpu = cpu_of(rq);
    
        // 如果rq的cpu是当前cpu
    	if (cpu == smp_processor_id()) {
    	    set_tsk_need_resched(curr);
    	    set_preempt_need_resched();
    	    return;
    	}
    
    	if (set_nr_and_not_polling(curr))
    	    smp_send_reschedule(cpu);
    	else
    	    trace_sched_wake_idle_without_ipi(cpu);
    }

如果当前cpu和rq的cpu一致, 则调用set_tsk_need_resched, 也就是设置task的thread_info的flag设置上TIF_NEED_RESCHED标志位

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1541
    static inline void set_tsk_need_resched(struct task_struct *tsk)
    {
    	set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);
    }


然后set_preempt_need_resched分平台的, 里面是汇编的, 没看懂

.. code-block:: c

    https://elixir.bootlin.com/linux/v4.15/source/arch/x86/include/asm/preempt.h#L55
    static __always_inline void set_preempt_need_resched(void)
    {
    	raw_cpu_and_4(__preempt_count, ~PREEMPT_NEED_RESCHED);
    }


default_wake_function/try_to_wake_up
============================================

ep_poll中, 等待有event发生的时候, 把默认的唤醒函数设置为default_wake_function, 而default_wake_function直接调用try_to_wake_up

try_to_wake_up是唤醒一个task的主要函数, 比如在epoll中如果有event发生, 那么会调用该函数去唤醒睡眠的task

调用路径: default_wake_function -> try_to_wake_up -> ttwu_queue -> ttwu_do_activate

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L1705

.. code-block:: c

    static void
    ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags,
    		 struct rq_flags *rf)
    {
    	int en_flags = ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK;
    
    	lockdep_assert_held(&rq->lock);
    
    #ifdef CONFIG_SMP
    	if (p->sched_contributes_to_load)
    		rq->nr_uninterruptible--;
    
    	if (wake_flags & WF_MIGRATED)
    		en_flags |= ENQUEUE_MIGRATED;
    #endif
    
    	ttwu_activate(rq, p, en_flags);
    	ttwu_do_wakeup(rq, p, wake_flags, rf);
    }


1. ttwu_activate最终也是调用enqueue_task函数, ttwu_activate -> activate_task -> enqueue_task

2. ttwu_do_wakeup则是调用check_preempt_curr去跟当前task抢占, check_preempt_curr最终调用到cfs中的check_preempt_wakeup

**这里注意的是:**

调用default_wake_function的上一层函数, 比如socket可读的时候, 调用

sock_def_readable -> wake_up_interruptible_sync_poll -> __wake_up_sync_key -> __wake_up_common这个路径中, 最终传递给default_wake_function的

wait_flag是__wake_up_sync_key中设置的


.. code-block:: c


    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L215
    #define wake_up_interruptible_sync_poll(x, m)					\
         __wake_up_sync_key((x), TASK_INTERRUPTIBLE, 1, (void *) (m))


    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L192
    void __wake_up_sync_key(struct wait_queue_head *wq_head, unsigned int mode,
    			int nr_exclusive, void *key)
    {
    	int wake_flags = 1; /* XXX WF_SYNC */
    
    	if (unlikely(!wq_head))
    		return;
    
    	if (unlikely(nr_exclusive != 1))
    		wake_flags = 0;
    
    	__wake_up_common_lock(wq_head, mode, nr_exclusive, wake_flags, key);
    }

可以看到, 在__wake_up_sync_key中, wait_flag一般是1, 因为一般传入的nr_exclusive是1来避免惊群

关于wait_flag, 有

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/sched.h#L1375

.. code-block:: c

    /*
     * wake flags
     */
    #define WF_SYNC		0x01		/* waker goes to sleep after wakeup */
    #define WF_FORK		0x02		/* child wakeup after fork */
    #define WF_MIGRATED	0x4		/* internal use, task got migrated */



ep_poll中休眠
===============

当调用ep_poll的时候, 会根据timeout让出cpu, 等待event的发生

.. code-block:: c

    // 省略了很多代码
    static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
    		   int maxevents, long timeout)
    {
    
        if (!ep_events_available(ep)) {
            
            // 这个for循环就是检查是否是被中断唤醒的了
            for (;;) {
                if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
                    timed_out = 1;
            }
        
        }
    
    }

主要函数是schedule_hrtimeout_range_clock, 而schedule_hrtimeout_range_clock则会调用schedule去让出cpu

.. code-block:: c

    /**
     * schedule_hrtimeout_range_clock - sleep until timeout
     * @expires:	timeout value (ktime_t)
     * @delta:	slack in expires timeout (ktime_t)
     * @mode:	timer mode, HRTIMER_MODE_ABS or HRTIMER_MODE_REL
     * @clock:	timer clock, CLOCK_MONOTONIC or CLOCK_REALTIME
     */
    int __sched
    schedule_hrtimeout_range_clock(ktime_t *expires, u64 delta,
    			       const enum hrtimer_mode mode, int clock)
    {
    
        struct hrtimer_sleeper t;
        
        /*
         * Optimize when a zero timeout value is given. It does not
         * matter whether this is an absolute or a relative time.
         */
        if (expires && *expires == 0) {
        	__set_current_state(TASK_RUNNING);
        	return 0;
        }
        
        /*
         * A NULL parameter means "infinite"
         */
        if (!expires) {
                // 调用schedule函数
        	schedule();
        	return -EINTR;
        }
        
        hrtimer_init_on_stack(&t.timer, clock, mode);
        hrtimer_set_expires_range_ns(&t.timer, *expires, delta);
        
        hrtimer_init_sleeper(&t, current);
        
        hrtimer_start_expires(&t.timer, mode);
        
        if (likely(t.task))
                // 调用schedule函数
        	schedule();
        
        hrtimer_cancel(&t.timer);
        destroy_hrtimer_on_stack(&t.timer);
        
        __set_current_state(TASK_RUNNING);
        
        return !t.task ? 0 : -EINTR;
    
    }

看到, 如果expires是NULL, 也就是无限睡眠的话, 则会调用schedule函数, 所以推测出, schedule函数会让出cpu的!!!


schedule/__schedule
=========================

**schedule(__schedule)函数就是做一次强制抢占操作的地方!!!!!!!!!!!!**

上面的resched_curr只是标识了curr需要被抢占, 那么某个地方, 判断到curr需要被抢占之后, 调用schedule或者__schedule

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3427

.. code-block:: c

    asmlinkage __visible void __sched schedule(void)
    {
    	struct task_struct *tsk = current;
    
    	sched_submit_work(tsk);
    	do {
    		preempt_disable();
                // 调用__schedule函数
    		__schedule(false);
    		sched_preempt_enable_no_resched();
    	} while (need_resched());
    }
    EXPORT_SYMBOL(schedule);

所以主要函数就是__schedule函数

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3287

.. code-block:: c

    // 省略了很多代码
    static void __sched notrace __schedule(bool preempt)
    {
    
        // prev就是当前cpu的runqueue中的当前task
        prev = rq->curr;

        // 看到schedule函数传入的preempt是false
        // 然后在ep_poll中把task状态设置为TASK_INTERRUPTIBLE, 该状态是大于0的
        // 所以会走到if的代码里面
        if (!preempt && prev->state) {
            // 如果此时当前的task有信号发生, 则直接当前task为TASK_RUNNINg
            if (unlikely(signal_pending_state(prev->state, prev))) {
            	prev->state = TASK_RUNNING;
            } else {

                // 看到unlikely标志, 说一般都走这里
                // 也就是把prev从红黑树中拿出来
                deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

            }
        }

        // 选择下一个task
        next = pick_next_task(rq, prev, &rf);
        
        if (likely(prev != next)) {
            rq->nr_switches++;
            rq->curr = next;

            rq = context_switch(rq, prev, next, &rf);
        
        }
    
    }

所以, ep_poll中休眠最终的调用是schedule函数, 该函数是把当前的task给抢占出去, 选择下一个task去运行, 也就是主动让出cpu时间了

1. 传入的preempt参数是什么意思, 没太明白

2. 如果task不是TASK_RUNNING状态(0x0000), 并且传入的preempt是false, 则触发deactivate_task

   deactivate_task会调用到dequeue_task去把task从红黑树移除

3. 调用pick_next_task选择下一个task, pick_next_task已经把选出来的next设置为cfs_rq->curr了, cfs_rq->curr = next

4. 然后在if (prev != next)的判断下, 把rq->curr设置为选出来的next, rq->curr = next, 所以此时rq->curr = cfs_rq = next

5. context_switch, 做一些寄存器切换等操作


dequeue_task/dequeue_task_fair
===================================

在之前epoll休眠的流程中, 可以看到, 调用了schedule函数之后, 由于设置了task的状态(task->state)为TASK_INTERRUPTIBLE, 则

schedule函数调用的__schedule函数, 会调用deactivate_task去调用到dequeue_task函数, 在cfs中, dequeue_task被指向函数dequeue_task_fair

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L5262

.. code-block:: c


    /*
     * The dequeue_task method is called before nr_running is
     * decreased. We remove the task from the rbtree and
     * update the fair scheduling stats:
     */
    static void dequeue_task_fair(struct rq *rq, struct task_struct *p, int flags)
    {
    	struct cfs_rq *cfs_rq;
        // 传入的task的se对象
    	struct sched_entity *se = &p->se;
    	int task_sleep = flags & DEQUEUE_SLEEP;
    
    	for_each_sched_entity(se) {
    	    cfs_rq = cfs_rq_of(se);
            // 移除操作函数
    	    dequeue_entity(cfs_rq, se, flags);
    
    	    /*
    	     * end evaluation on encountering a throttled cfs_rq
    	     *
    	     * note: in the case of encountering a throttled cfs_rq we will
    	     * post the final h_nr_running decrement below.
    	    */
    	    if (cfs_rq_throttled(cfs_rq))
    	    	break;
    	    cfs_rq->h_nr_running--;
    
    	    /* Don't dequeue parent if it has other entities besides us */
    	    if (cfs_rq->load.weight) {
    	    	/* Avoid re-evaluating load for this entity: */
    	    	se = parent_entity(se);
    	    	/*
    	    	 * Bias pick_next to pick a task from this cfs_rq, as
    	    	 * p is sleeping when it is within its sched_slice.
    	    	 */
    	    	if (task_sleep && se && !throttled_hierarchy(cfs_rq))
    	    		set_next_buddy(se);
    	    	break;
    	    }
    	    flags |= DEQUEUE_SLEEP;
    	}
    
    	for_each_sched_entity(se) {
    	    cfs_rq = cfs_rq_of(se);
    	    cfs_rq->h_nr_running--;
    
    	    if (cfs_rq_throttled(cfs_rq))
    	    	break;
    
    	    update_load_avg(cfs_rq, se, UPDATE_TG);
    	    update_cfs_group(se);
    	}
    
    	if (!se)
    	    sub_nr_running(rq, 1);
    
    	hrtick_update(rq);
    }

除了dequeue_entity函数, 其他流程, 恩~~~不太清楚


pick_next_task/pick_next_task_fair
========================================

pick_next_task这个函数将会调用到cfs中的pick_next_task_fair

该函数中, 如果开启了CONFIG_FAIR_GROUP_SCHED, 也就是开启了组调度, 那么流程复杂一点(默认组调度是开启的)

然后这里只是看流程, 所以直接去看simple代码块的流程, simple则是没有配置组调度时候最简单的选择过程

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L6619

.. code-block:: c

    static struct task_struct *
    pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
    {
    
    
    #ifdef CONFIG_FAIR_GROUP_SCHED
        // 省略代码, 其中包括配置了组调度的流程
    simple:
    #endif
        // simple是最简单情况下选择的流程

        // 把当前的task放入cfs红黑树中
        // 之前我们把当前task给dequeue了
        put_prev_task(rq, prev);
        
        do {
            // 挑选下一个task
            se = pick_next_entity(cfs_rq, NULL);
            // 把下一个task设置为curr
            set_next_entity(cfs_rq, se);
            cfs_rq = group_cfs_rq(se);
        } while (cfs_rq);
        
        p = task_of(se);

        // 还省略了很多代码


    
    
    }


1. pick_next_entity, 是选择合适的task来运行, 不一定是leftmost

2. put_prev_task, 把prev, 也就是传入的task, 重新加入红黑树, 因为在__schedule中, 我们移除了task

3. set_next_entity, 把1中选择的task, 设置为cfs_rq->curr


pick_next_entity
===================

参考 [22]_

主要流程是:

1. leftmost和curr谁小, 谁小选谁, 这里保存到se

2. 如果cfs_rq中配置了需要跳过某个task, cfs_rq->skip, 并且cfs_rq->skip == se, 那么需要去选择一个"备胎":

   如果1中se是curr, 也就是curr小于leftmost, 那么second=leftmost
   
   否则second = cfs_rq->next, 但是如果cfs_rq->next不存在或者curr->vruntime < cfs_rq->next->vruntime

   那么second = curr

3. 接2, 然后调用wakeup_preempt_entity(second, leftmost), 如果lefmost大于second

   那么se = second

4. 接着, 调用wakeup_preempt_entity(cfs_rq->last, se), 如果last小于se, 则se = last

5. 接着调用wakeup_preempt_entity(cfs_rq->next, se), 如果next小于se, 则se = next

6. 最后, 因为我们已经判断next和last, 需要把cfs_rq->next, cfs_rq->last给置空

所以, 简单总结起来就是, curr, leftmost, next, last, 选一个最小的, 先选leftmost, 如果leftmost是需要skip, 那么第二

优先的应该是next, 最后和next/last做比较

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L4240

.. code-block:: c

    /*
     * Pick the next process, keeping these things in mind, in this order:
     * 1) keep things fair between processes/task groups
     * 2) pick the "next" process, since someone really wants that to run
     * 3) pick the "last" process, for cache locality
     * 4) do not run the "skip" process, if something else is available
     */
    static struct sched_entity *
    pick_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *curr)
    {
    	struct sched_entity *left = __pick_first_entity(cfs_rq);
    	struct sched_entity *se;
    
    	/*
    	 * If curr is set we have to see if its left of the leftmost entity
    	 * still in the tree, provided there was anything in the tree at all.
    	 */
    	if (!left || (curr && entity_before(curr, left)))
    	    left = curr;
    
    	se = left; /* ideally we run the leftmost entity */
    
    	/*
    	 * Avoid running the skip buddy, if running something else can
    	 * be done without getting too unfair.
    	 */
    	if (cfs_rq->skip == se) {
    	    struct sched_entity *second;
    
    	    if (se == curr) {
    	    	second = __pick_first_entity(cfs_rq);
    	    } else {
    	    	second = __pick_next_entity(se);
    	    	if (!second || (curr && entity_before(curr, second)))
    	    		second = curr;
    	    }
    
    	    if (second && wakeup_preempt_entity(second, left) < 1)
    		se = second;
    	}
    
    	/*
    	 * Prefer last buddy, try to return the CPU to a preempted task.
    	 */
    	if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1)
    	    se = cfs_rq->last;
    
    	/*
    	 * Someone really wants this to run. If it's not unfair, run it.
    	 */
    	if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1)
    	    se = cfs_rq->next;
    
        // 注意, 判断过next和last之后, 要把next, last置空
    	clear_buddies(cfs_rq, se);
    
    	return se;
    }



put_prev_task/set_next_entity
=================================

put_prev_task会调用到cfs中的put_prev_task_fair, 作用则是调用__enqueue_entity去把传入的prev加入到红黑树

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L6754
    static void put_prev_task_fair(struct rq *rq, struct task_struct *prev)
    {
    	struct sched_entity *se = &prev->se;
    	struct cfs_rq *cfs_rq;
    
    	for_each_sched_entity(se) {
    		cfs_rq = cfs_rq_of(se);
                // 对每一个循环的se调用put_prev_entity
    		put_prev_entity(cfs_rq, se);
    	}
    }

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L4292
    static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev)
    {
    	/*
    	 * If still on the runqueue then deactivate_task()
    	 * was not called and update_curr() has to be done:
    	 */
        // 经过dequeue_task之后, 传入的task应该不会在rq上了, 也就是on_rq=0
    	if (prev->on_rq)
    	    update_curr(cfs_rq);
    
    	/* throttle cfs_rqs exceeding runtime */
    	check_cfs_rq_runtime(cfs_rq);
    
    	check_spread(cfs_rq, prev);
    
    	if (prev->on_rq) {
    	    update_stats_wait_start(cfs_rq, prev);
    	    /* Put 'current' back into the tree. */
    	    __enqueue_entity(cfs_rq, prev);
    	    /* in !on_rq case, update occurred at dequeue */
    	    update_load_avg(cfs_rq, prev, 0);
    	}
    	cfs_rq->curr = NULL;
    }

而set_next_entity是cfs的函数, 是把选出来的next设置到cfs_rq->curr

注意的是, 红黑树上的task和cfs->curr是互斥的, 也就是说

**如果一个task选出来设置为curr, 那么得从红黑树中移除**

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L4198
    static void
    set_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    	/* 'current' is not kept within the tree. */
        // 传入的se之前是在红黑树的leftmost的, 也是经过enqueue_task的, 也就是on_rq=1
    	if (se->on_rq) {
    	    /*
    	     * Any task has to be enqueued before it get to execute on
    	     * a CPU. So account for the time it spent waiting on the
    	     * runqueue.
    	     */
    	    update_stats_wait_end(cfs_rq, se);
            // 出队
    	    __dequeue_entity(cfs_rq, se);
    	    update_load_avg(cfs_rq, se, UPDATE_TG);
    	}
    
    	update_stats_curr_start(cfs_rq, se);
        // 把传入的task设置为curr
    	cfs_rq->curr = se;
    
        // 后面代码先省略
    }

scheduler_tick
=================

这个函数是每一个时钟周期都会去被调用, 主要是看有没有需要抢占的呀, 然后做一下load_balance

注意的是, 这里会调用cfs的task_tick函数, 也就是task_tick_fair


.. code-block:: c

    void scheduler_tick(void)
    {
    	int cpu = smp_processor_id();
    	struct rq *rq = cpu_rq(cpu);
    	struct task_struct *curr = rq->curr;
    	struct rq_flags rf;
    
    	sched_clock_tick();
    
    	rq_lock(rq, &rf);
    
    	update_rq_clock(rq);
        // 调用cfs的task_tick
    	curr->sched_class->task_tick(rq, curr, 0);
    	cpu_load_update_active(rq);
    	calc_global_load_tick(rq);
    
    	rq_unlock(rq, &rf);
    
    	perf_event_task_tick();
    
    #ifdef CONFIG_SMP
    	rq->idle_balance = idle_cpu(cpu);
        // 这里是发一个软中断
        // 然后软中断的handler函数就是去处理load_balance了
    	trigger_load_balance(rq);
    #endif
    	rq_last_tick_reset(rq);
    } 


task_tick_fair
=================

这里的主要功能(也可以说是周期性调度的工作), 是判断curr的时间片是否用完, 如果用完了, 那么调用resched_curr设置curr为被抢占状态, 如果curr

已经被设置为被抢占状态了, 则直接退出

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/fair.c#L9423

.. code-block:: c

    static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
    {
    	struct cfs_rq *cfs_rq;
    	struct sched_entity *se = &curr->se;
    
    	for_each_sched_entity(se) {
    		cfs_rq = cfs_rq_of(se);
                // 主要处理在这里!!!!!!!!!!!!!!!!!!
    		entity_tick(cfs_rq, se, queued);
    	}
    
    	if (static_branch_unlikely(&sched_numa_balancing))
    		task_tick_numa(rq, curr);
    }


主要处理还是在entity_tick函数中


.. code-block:: c

    static void
    entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
    {
    	/*
    	 * Update run-time statistics of the 'current'.
    	 */
        // 更新一下curr->vruntime
    	update_curr(cfs_rq);
    
    	/*
    	 * Ensure that runnable average is periodically updated.
    	 */
    	update_load_avg(cfs_rq, curr, UPDATE_TG);
    	update_cfs_group(curr);
    
    #ifdef CONFIG_SCHED_HRTICK
        // 里面的代码先省略
    #endif
    
        // 如果cfs中有超过一个task在运行
    	if (cfs_rq->nr_running > 1)
            调用check_preempt_tick
    	    check_preempt_tick(cfs_rq, curr);
    }


check_preempt_tick
======================

判断当前curr是否应该被抢占掉, 判断条件是:

1. curr的运行时间delta_exec, 大于被分配的时间片ideal_runtime, 那么表示curr的时间片是用完了, 则调用resched_curr, 然后退出

2. 没用完, 但是curr没达到最小运行时间sysctl_sched_min_granularity, 则绝对不能被抢占, 退出

3. 如果curr时间片没用完, 并且大于sysctl_sched_min_granularity, 此时需要判断leftmost能不能抢占curr
   
   如果curr->vruntime和leftmost->vruntime的差值delta, 大于curr的被分配时间片, 则leftmost应该抢占掉curr

**和check_preempt_wakeup(check_preempt_curr)的不同是, 前者是计算唤醒的task(包括新建的)的vruntime是否小于curr并且小于curr+gran**

.. code-block:: c

    static void
    check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
    {
    	unsigned long ideal_runtime, delta_exec;
    	struct sched_entity *se;
    	s64 delta;
    
        // 这里有一个时间的计算, 是task被分配的时间
    	ideal_runtime = sched_slice(cfs_rq, curr);
        // 计算运行时间
    	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
        // 如果运行时间大于被分配的时间
    	if (delta_exec > ideal_runtime) {

            // 设置抢占标志位
    	    resched_curr(rq_of(cfs_rq));
    	    /*
    	     * The current task ran long enough, ensure it doesn't get
    	     * re-elected due to buddy favours.
    	     */
            // 清除last, next
    	    clear_buddies(cfs_rq, curr);
    	    return;
    	}
    
    	/*
    	 * Ensure that a task that missed wakeup preemption by a
    	 * narrow margin doesn't have to wait for a full slice.
    	 * This also mitigates buddy induced latencies under load.
    	 */
        // 如果运行时间不足最小的运行时间, 则绝对不能被抢占
    	if (delta_exec < sysctl_sched_min_granularity)
    	    return;
    
        // 选出最左节点
        se = __pick_first_entity(cfs_rq);
    	delta = curr->vruntime - se->vruntime;
    
    	if (delta < 0)
    	    return;
    
        // 这里delta > ideal_runtime
        // 基本上是判断leftmost < curr, 并且leftmost < curr - gran
    	if (delta > ideal_runtime)
    	    resched_curr(rq_of(cfs_rq));
    }


ideal_runtime
================

这里的ideal_runtime, 是curr被分配出来的运行时间, 是通过sched_slice计算

而之前说明到, sched_slice的计算是根据当前运行的task计算出来的, 也就是一个task理想运行时间是

取一个基准slice, 然后取该task的load在整个cfs_rq的load的比例

base_slice = sysctl_sched_latency 或者 nr_running * sysctl_sched_min_granularity

ideal_runtime = base_slice * (se->load / cfs_rq->load)

运行的时间的计算
===================

当前task已经运行了多久, 是当前累计运行(sum_exec_runtime)和上一次累计运行时间(prev_sum_exec_runtime)的差值

也就是t1这个task, prev_sum_exec_runtime=0, sum_exec_runtime=10, 然后被抢占, 然后下一次再选择t1的时候, prev_sum_exec_runtime记录下

sum_exec_runtime, prev_sum_exec_runtime = sum_exec_runtime = 10, 然后t1一直运行, sum_exec_runtime一直增加, 有

sum_exec_runtime = 20, prev_sum_exec_runtime=10, 此时进入check_preempt_tick, 那么delta_exec = 20 - 10 = 10

prev_sum_exec_runtime的赋值是在 *__schedule -> pick_next_task - >pick_next_fair -> set_next_entity* 中

一旦一个task被选择, 那么把上一次运行的时间保存到prev_sum_exec_runtime上


.. code-block:: c

    static void
    set_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
        // 省略代码

        // 被选择为下一个运行的task, 保存下上一次运行时间
    	se->prev_sum_exec_runtime = se->sum_exec_runtime;
    }

几个slice的计算总结
======================

calc_delta_fair
-----------------

这个函数是做这样的计算, 第一个参数是delta, 第二个参数是se, 

1. 如果se->load == NICE_0_LOAD, 那么new_delta = delta

2. 否则根据se->load和NICE_0_LOAD的比例, 取delta
   
   也就是new_delta = delta * (NICE_0_LOAD / se->load)

.. code-block:: c

    static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
    {
    	if (unlikely(se->load.weight != NICE_0_LOAD))
            // __calc_delta是计算delta * (NICE_0_LOAD / se->load)
            delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    
    	return delta;
    }



sched_slice
-------------

这个函数是计算指定的task在cfs_rq上, 应该得到多少时间, 计算方式是: 一个基准时间片, 然后取task在cfs_rq的load的比例

.. code-block:: c

    static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);
    
    	for_each_sched_entity(se) {
    	    struct load_weight *load;
    	    struct load_weight lw;
    
    	    cfs_rq = cfs_rq_of(se);
    	    load = &cfs_rq->load;
    
    	    if (unlikely(!se->on_rq)) {
    	    	lw = cfs_rq->load;
    
    	    	update_load_add(&lw, se->load.weight);
    	    	load = &lw;
    	    }
    	    slice = __calc_delta(slice, se->load.weight, load);
    	}
    	return slice;
    }

首先, __sched_period返回的是基准的slice, slice = nr_running * sysctl_sched_min_granularity if nr_running > sched_nr_latency) else sysctl_sched_latency

然后调用__calc_delta直接计算slice = slice * (se->load / cfs_rq->load)

**也就是一个task被分配多少时间是: 取一个基准的slice, task分到多少取决于其在cfs_rq上所占的比例**


sched_vslice
---------------

这个是用在补偿新建task的vruntime的

**补偿的值是: task被分配的时间t, 然后取决于其load和NICE_0_LOAD的比例**

t  = sched_slice = slice * (se->load / cfs_rq-load)

vt = t * (NICE_0_LOAD / se->load)

.. code-block:: c

    static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    	return calc_delta_fair(sched_slice(cfs_rq, se), se);
    } 

wakeup_preempt_entity/wakeup_gran
===================================

这个是在check_preempt_wakeup中, 判断传入的task(se)是否抢占掉curr

抢占的条件是, task->vruntime < curr->vruntime, 并且task->vruntime < curr->vruntime - wakeup_gran

wakeup_gran = sysctl_sched_wakeup_granularity * (NICE_0_LOAD / se->load)

.. code-block:: c

    static unsigned long
    wakeup_gran(struct sched_entity *curr, struct sched_entity *se)
    {
        // 基准值
    	unsigned long gran = sysctl_sched_wakeup_granularity;
    
        // 调用之前的calc_delta_fair
    	return calc_delta_fair(gran, se);
    }



TIF_NEED_RESCHED
====================

参考 [20]_

上面那么多的流程, 可以看到, **__schedule函数(其实不是schedule函数)才是强制做一次抢占(pick_next_entity, put_prev_entity, set_next_entity)的地方**

那么什么时候__schedule才会被调用呢? 可以看看__schedule函数的注释

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3287

.. code-block:: c

    /*
     * __schedule() is the main scheduler function.
     *
     * The main means of driving the scheduler and thus entering this function are:
     *
     *   1. Explicit blocking: mutex, semaphore, waitqueue, etc.
     *
     *   2. TIF_NEED_RESCHED flag is checked on interrupt and userspace return
     *      paths. For example, see arch/x86/entry_64.S.
     *
     *      To drive preemption between tasks, the scheduler sets the flag in timer
     *      interrupt handler scheduler_tick().
     *
     *   3. Wakeups don't really cause entry into schedule(). They add a
     *      task to the run-queue and that's it.
     *
     *      Now, if the new task added to the run-queue preempts the current
     *      task, then the wakeup sets TIF_NEED_RESCHED and schedule() gets
     *      called on the nearest possible occasion:
     *
     *       - If the kernel is preemptible (CONFIG_PREEMPT=y):
     *
     *         - in syscall or exception context, at the next outmost
     *           preempt_enable(). (this might be as soon as the wake_up()'s
     *           spin_unlock()!)
     *
     *         - in IRQ context, return from interrupt-handler to
     *           preemptible context
     *
     *       - If the kernel is not preemptible (CONFIG_PREEMPT is not set)
     *         then at the next:
     *
     *          - cond_resched() call
     *          - explicit schedule() call
     *          - return from syscall or exception to user-space
     *          - return from interrupt-handler to user-space
     *
     * WARNING: must be called with preemption disabled!
     */

也就是说, __schedule函数是主要的调度函数, 进入该函数意味着:

1. 显式阻塞, 也就是主动放弃cpu, 包括锁, 信号量, waitqueue等待

2. TIF_NEED_RESCHED表示被设置上了, 也就是应该去做一次抢占操作, 比如在时钟周期中调用的scheduler_tick函数

   会设置上这个标志位

3. 如果新建的task被加入到runqueue, 然后应该抢占掉curr的话, 可以对curr设置这个TIF_NEED_RESCHED标志位

   然后, schedule函数会最近一次, 下面任一一种情况下被调用, 下面的情况没怎么看懂


所以, 在resched_curr函数中, 只是对task结构设置了TIF_NEED_RESCHED标志位d的地方, 看起来两者并没有什么关联.

  *Even though this flag is set at this point, the task is not going to be preempted yet*
  
  --- 参考 20

也就是说, 在resched_curr中对task设置TIF_NEED_RESCHED标志位之后, 并没有实际上进行一次强行抢占

  *This is because preemption happens at specific points such as exit of interrupts. If the flag is set because the timer interrupt (scheduler decided) decided that something of higher priority needs CPU now and sets TIF_NEED_RESCHED, then at the exit of the timer interrupt (interrupt exit path), TIF_NEED_RESCHED is checked, and because it is set – schedule() is called causing context switch to happen to another process of higher priority, instead of just returning to the existing process that the timer interrupted normally would*
  
  --- 参考 20

也就是这个标志位为设置的时候, 只是说明rq->curr需要被抢占掉, 把cpu让给其他task. 而在一个时间周期(默认1ms)内的处理函数scheduler_tick也是设置TIF_NEED_RESCHED标志位而已. 

然后在某个时间点会调用schedule/__schedule, 去强行抢占掉curr, 比如epoll中休眠的时候, 调用了schedule函数, 让出cpu

或者, 某个地方会去校验task的TIF_NEED_RESCHED标志位, 如果被设置了, 则调用schedule(实际是__schedule)函数去强行切换task.

**那什么时候去校验TIF_NEED_RESCHED标志为然后调用schedule函数呢?**

*If the tick interrupt happened user-mode code was running, then in somewhere in the interrupt exit path for x86, this call chain calls schedule ret_from_intr –> reint_user –> prepare_exit_to_usermode. Here the need_reched flag is checked, and if true schedule() is called.*

比如x86架构下, 当时钟中断发生的时候, 用户代码正在运行, 那么用户代码被中断, 然后时钟中断处理完成, 退回用户态的时候, prepare_exit_to_usermode函数将会去校验TIF_NEED_RESCHED标志位


https://elixir.bootlin.com/linux/v4.15/source/arch/x86/entry/common.c#L182

.. code-block:: c

    /* Called with IRQs disabled. */
    __visible inline void prepare_exit_to_usermode(struct pt_regs *regs)
    {
        // 当前task的thread_info结构
    	struct thread_info *ti = current_thread_info();
    	u32 cached_flags;
    
    	addr_limit_user_check();
    
    	lockdep_assert_irqs_disabled();
    	lockdep_sys_exit();
    
        // 拿到task的thread_info的标志位
    	cached_flags = READ_ONCE(ti->flags);
    
    	if (unlikely(cached_flags & EXIT_TO_USERMODE_LOOP_FLAGS))
            // 传入thread_info的标志位, 然后去处理
            exit_to_usermode_loop(regs, cached_flags);
    
        // 省略代码
    }

处理抢占

https://elixir.bootlin.com/linux/v4.15/source/arch/x86/entry/common.c#L137

.. code-block:: c

    static void exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags)
    {
        /*
         * In order to return to user mode, we need to have IRQs off with
         * none of EXIT_TO_USERMODE_LOOP_FLAGS set.  Several of these flags
         * can be set at any time on preemptable kernels if we have IRQs on,
         * so we need to loop.  Disabling preemption wouldn't help: doing the
         * work to clear some of the flags can sleep.
         */
        while (true) {
            /* We have work to do. */
            local_irq_enable();
            
            if (cached_flags & _TIF_NEED_RESCHED)
                // 校验标志位之后去调用schedule函数
                // 做一次强制抢占
                schedule();
    
            if (cached_flags & _TIF_UPROBE)
            	uprobe_notify_resume(regs);
            
            // 这里还校验了是否有信号需要处理
            /* deal with pending signal delivery*/
            if (cached_flags & _TIF_SIGPENDING)
            	do_signal(regs);
            
            // 省略代码
    
        }
        
    }


*For return from interrupt to kernel mode, things are a bit different (skip this para if you think it’ll confuse you).

This feature requires kernel preemption to be enabled. The call chain doing the preemption is: ret_from_intr –> reint_kernel –> preempt_schedule_irq (see arch/x86/entry/entry_64.S) which calls schedule. Note that, for return to kernel mode, I see that preempt_schedule_irq calls schedule anyway whether need_resched flag is set or not, this is probably Ok but I am wondering if need_resched should be checked here before schedule is called. Perhaps it would be an optimiziation to avoid unecessarily calling schedule*


也可能是从中断回到内核态(这个情况我没清楚), 也就是说, 一个中断也会回到内核态, 但是需要启用, 也就是配置CONFIG_PREEMPT=y,

会调用到preempt_schedule_irq, 这个调用也直接调用schedule函数而不管TIF_NEED_RESCHED是否被设置上. 作者没明白为什么, 我也没明白为什么.

后面还有作者的猜测, 这里就不贴出来了

但其实在4.15的代码中, preempt_schedule_irq函数其实是去检查TIF_NEED_RESCHED标志位的

下面是preempt_schedule_irq函数, 里面直接调用了__schedule函数, 注意的是, 传参是false而不是schedule函数调用__schedule的时候传入的true, 传入的true和false有什么不同? 不清楚

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3605

.. code-block:: c

    asmlinkage __visible void __sched preempt_schedule_irq(void)
    {
    	enum ctx_state prev_state;
    
    	/* Catch callers which need to be fixed */
    	BUG_ON(preempt_count() || !irqs_disabled());
    
    	prev_state = exception_enter();
    
    	do {
    		preempt_disable();
    		local_irq_enable();
    		__schedule(true);
    		local_irq_disable();
    		sched_preempt_enable_no_resched();
    	} while (need_resched());
    
    	exception_exit(prev_state);
    }

    // 函数need_resched就是校验TIF_NEED_RESCHED标志位的地方
    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1611
    static __always_inline bool need_resched(void)
    {
        // 这个函数就是校验的地方
    	return unlikely(tif_need_resched());
    }


