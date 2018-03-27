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

CFS中的vruntime
==================

CFS中用红黑树存储task, 红黑树的key是task(sched_entity)中的vruntime属性的值. CFS会从红黑树中拿到下一个task, 而下一个task的是红黑树中的最左叶节点(left_most)

而CFS中会把最左叶节点给缓存起来的, 也就是查找的时候直接访问而不是要经过一个log(n)的查找过程.

vruntime的是这样子的, 每当从红黑树拿到下一个task去运行, 那么该task的vruntime就变大, 也就是其被放入到右子节点中, 然后剩下的vruntime比较下的task

就有机会运行了. 这样保证了某个task一定会被运行, 比如a, b两个task, a的runtime是10, b的是30, 然后a运行, 假设a的vruntime每次加5, 那么a运行了

6次之后, b就会被选中运行.

优先级高的task, vruntime的增加会比较慢, 而优先级低的task, 其vruntime会增加得比较快, 保证优先级高的运行时间更多. 上面的a和b两个task, a优先级高, 所以其vruntime

增加得比较慢, 一次加5. 所以a会比b运行次数(和时间)都会比b多.

vruntime增加的值则是公共task自身的优先级(也就是权重)计算出来的.

这里的vruntime是虚拟的运行时间, 在cfs中, 还保存了实际总运行的cpu时间, sum_exec_runtime, 所以两者是不同的. vruntime则是用来选择下一个task的, 而sum_exec_runtime

则是真实的已经运行过的cpu时间

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



当一个task处于运行状态的时候, 内核调用enqueue_task, 该函数的作用是把指定的task加入到cpu的runqueue里面(优先级插入?)

*Each CPU(core) in the system has its own runqueue, and any task can be included in at most one runqueue;*

*A process scheduler’s job is to pick one task from a queue and assign it to run on a respective CPU(core).*


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

这个函数是初始化task中的调度属性的地方

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
        // 初始化各种属性为0, 注意看vruntime和sum_exec_runtime都被设置为0
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


fair_sched_class->task_fork
==============================

sched_fork中, 最后调用fair_sched_class中的task_fork函数

在fair.c中, 该函数被定义为task_fork_fair

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
    	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
    		/*
    		 * Upon rescheduling, sched_class::put_prev_task() will place
    		 * 'current' within the tree based on its new key value.
    		 */
    		swap(curr->vruntime, se->vruntime);
    		resched_curr(rq);
    	}
    
    	se->vruntime -= cfs_rq->min_vruntime;
    	rq_unlock(rq, &rf);
    }

place_entity
---------------

这个函数会对task的vruntime进行补偿, 对新的task和io唤醒的task都有对应的补偿

补偿的基础是min_vruntime

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
        // 并且设置了
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

sched_features的START_DEBIT位：规定新进程的第一次运行要有延迟。

1. 补偿的基础是min_vruntime

2. 如果是新建task, 并且规定新建的task第一次启动需要延迟, 则调用sched_vslice计算补偿, vruntime += sched_vslice

3. 如果不是新建并且设置了GENTLE_FAIR_SLEEPERS, 则表示是io唤醒需要补偿, 这里是减少, 上面2是增加vruntime -= thresh

4. 最后, 取补偿vruntime和se自己的vruntime的最大值


小结
-------

1. update_curr是核心的更新vruntime的函数, 更新的是cfs中当前task的vruntime, 所以传参才只有cfs_rq, 后面说

2. place_entity函数查看参考 [16]_, 是对task的vruntime的补偿操作

3. sysctl_sched_child_runs_first配置是说是否配置子线程在父线程之前运行, 如果是, 并且父线程大于子线程(entity_before函数), 那么交换两个
   线程的vruntime, 然后调用resched_curr, 这部分参考 [16]_

4. 最后, 为什么se->vruntime要减去min_vruntime, 不清楚


wake_up_new_task
===================

这个函数是_do_fork中唤醒新task结构的地方

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

clone流程总结
==================================

所以, 总结下来, pthread_create的时候, 子线程会继承父线程调度的参数, 包括调度策略和load_weight, 然后

copy_process中调用sched_fork去初始化调度相关的参数:

1. 调用__sched_fork, 把vruntime和sum_exec_runtime设置为0

2. 调用fair_sched_class->task_fork_fair, 对task的vruntime进行补偿

然后wake_up_new_task则会:

1. 设置task的状态为TASK_RUNNING, 然后如果在SMP架构下, 需要再次设置cpu(因为1. cpu_allowed可能有变化 2. 之前选择的cpu可能不可用了)

2. 调用activate_task函数去调用相关调度类的enqueue_task函数, 把task加入到cfs自己的红黑树中



try_to_wake_up
==================

try_to_wake_up是唤醒一个task的主要函数, 比如在epoll中如果有event发生, 那么会调用该函数去唤醒睡眠的task

调用路径: try_to_wake_up -> ttwu_queue -> ttwu_do_activate


.. code-block:: c


    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L1705
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


1. ttwu_activate是去把task加入到红黑树中, 也就是调用enqueue_task函数, ttwu_activate -> activate_task -> enqueue_task

2. ttwu_do_wakeup则是调用check_preempt_curr去跟当前task抢占, check_preempt_curr最终调用到cfs中的check_preempt_wakeup


ep_poll
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


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3427
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


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3287
    // 省略了很多代码
    static void __sched notrace __schedule(bool preempt)
    {
    
        // prev就是当前cpu的runqueue中的当前task
        prev = rq->curr;

        // 看到schedule函数传入的preempt是false
        // 然后在ep_poll中把task状态设置为TASK_INTERRUPTIBLE, 该状态是大于0的
        // 所以会走到if的代码里面
        if (!preempt && prev->state) {
            // 如果此时有信号发生, 则直接设置prev的状态为TASK_RUNNING状态
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
        
            rq = context_switch(rq, prev, next, &rf);
        
        }
    
    }

所以, ep_poll中休眠最终的调用是schedule函数, 该函数是进行一次调度操作, 作用:

1. 如果task不是TASK_RUNNING状态(0x0000), 并且传入的preempt是false, 则触发deactivate_task
   deactivate_task会调用到dequeue_task去把task从红黑树移除

2. 选择下一个task


----

几个重要的函数
=================

1. update_curr, 更新当前cfs->curr的vruntime, 这个函数在很多地方都会被调用到

2. task_for_fair/place_entity, 对新建的task的vruntime进行补偿, 补偿的函数是place_entity, 这两个之前说过

3. enqueue_task/dequeue_task, 前者把task加入到cfs的红黑树中, 后者把task移除

4. check_preempt_curr, 去进行抢占的操作

5. schedule, 该函数去选择下一个task去运行

update_curr
===============

更新cfs中当前运行的task的vruntime属性

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
    
        // task的总运行时间增加delta
    	curr->sum_exec_runtime += delta_exec;
    	schedstat_add(cfs_rq->exec_clock, delta_exec);
    
        // 计算当前task的vruntime
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


1. calc_delta_fair的代码流程是

如果curr.nice != NICE_0_LOAD, 则curr−>vruntime += delta_exec * (NICE_0_LOAD/curr−>se−>load.weight)

如果curr.nice == NICE_0_LOAD, 则curr−>vruntime+=delta

也就是如果当前task的优先级是默认的0, 也就是120(0), 那么task的vruntime的增量则是delta值, 否则是delta乘以其优先级和默认优先级之间load weight的比例

所以, 优先级越高, load weight越大, 则delta越小, 则vruntime的变大得越慢.


2. update_min_vruntime, 这个函数是更新cfs_rq中, 最小的vruntime的, 之所以还需要一个cfs_rq的最小vruntime, 是因为插入红黑树的时候, 限制最小的vruntime值至少
   大于该值. 比如新建一个task, 设置其vruntime=0(在copy_process中), 么那么它在相当长的时间内都会保持抢占CPU的优势, 这样就不好, 所以需要min_vruntime去限制
   最小大小(参考 [16]_)

update_min_vruntime
=====================

比对当前task和红黑树中保存的最左叶节点两者的vruntime, 谁大设置为cfs->min_vruntime

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

主要流程是, 比对curr->vruntime和se-vruntime之间的最小值为vruntie, 然后min_vruntime = max(min_vruntime, vruntime)

1. 如果curr和se都存在, 那么min_vruntime = max(min_vruntime, min(curr->vruntime, se->vruntime))

2. 如果curr不存在而se存在, 那么min_vruntime = max(min_vruntime, se->vruntime)

3. 如果curr存在而se不存在, 那么min_vruntime = max(min_vruntime, curr->vruntime)

4. 如果curr和se都不存在,   那么min_vruntime = max(min_vruntime, min_vruntime)


enqueue_task/enqueue_task_fair
================================

之前的try_to_wake_up函数和wake_up_new_task函数都会调用到activate_task, activate_task基本上就是调用enqueue_task去把目标task给加入到cfs的红黑树中

enqueue_task在cfs中指向enqueue_task_fair函数

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
    
    	for_each_sched_entity(se) {
    		if (se->on_rq)
    			break;
    		cfs_rq = cfs_rq_of(se);
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




