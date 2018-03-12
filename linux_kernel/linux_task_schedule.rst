########################
内核中的task结构以及调度
########################

参考:

.. [1] https://www.lifewire.com/what-is-linux-2201940

.. [2] https://unix.stackexchange.com/questions/364660/are-threads-implemented-as-processes-on-linux
 
.. [3] https://tampub.uta.fi/bitstream/handle/10024/96864/GRADU-1428493916.pdf
 
.. [4] https://randu.org/tutorials/threads/

.. [5] https://www.cnblogs.com/wangzahngjun/p/4977425.html (slab的简单解释)

.. [6] https://www.ibm.com/developerworks/cn/linux/l-cn-slub/ (slub的简单解释)

.. [7] https://www.ibm.com/developerworks/linux/library/l-completely-fair-scheduler/


下面的内容绝大部分来自参考 [3]_, 也就是翻译一下

pthread/task
=============

内核的调度单位是task, 无论是进程还是线程, 都会映射到task结构中, 也就是lwp(Light Weight Process)

具体参考: glibc_nptl.rst


KThread
===========

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


调度的周期/策略
================

下面的参考都出自参考 [3]_, 基本上下面就是翻译了.

调度的发生是那时钟周期执行的, 内核中时钟周期是1/1000秒. 也就是每个时间周期内核都会去判断是否需要切换当前的task. 如果不需要切换task, 那么当前task则会运行下去/

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


task结构
=============

参考 [3]_




优先级
==========


.. code-block:: c

    int				prio;
    int				static_prio;
    int				normal_prio;
    unsigned int		rt_priority;
    const struct sched_class	*sched_class;
    struct sched_entity		se;


load weight
==============

task获取到多少的timeslice, 取决于优先级(调度策略), 但是具体到多少的timeslice, 或者说timeslice的大小, 取决于load weight.


调度类
==========

*The kernel decides, which tasks go to which scheduling classes based on their scheduling policy(SCHED_*) and calls the corresponding functions*

内核会根据task的属性去决定task的调度类, 然后调用调度类的指定函数. 这就是解耦了嘛

/kernel/sched/文件夹是调度的源码, 其中:

1. core.c中定义了调度类必须实现的一般性接口

2. fair.c实现了一般(normal)task的调度策略, 也就是CFS(Completely fair Scheduler), 也就是完全公平

3. rt.c实现了实时(real time)任务的调度策略

4. idle.c实现了空闲(idle)task的调度策略



当一个task处于运行状态的时候, 内核调用enqueue_task, 该函数的作用是把指定的task加入到cpu的runqueue里面(优先级插入?)

*Each CPU(core) in the system has its own runqueue, and any task can be included in at most one runqueue;*

*A process scheduler’s job is to pick one task from a queue and assign it to run on a respective CPU(core).*



比如a, b两个task都是使用A这个调度类, 则有:


