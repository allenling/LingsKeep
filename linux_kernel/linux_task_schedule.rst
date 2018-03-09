########################
内核中的task结构以及调度
########################

参考:

.. [1] https://www.lifewire.com/what-is-linux-2201940

.. [2] https://unix.stackexchange.com/questions/364660/are-threads-implemented-as-processes-on-linux
 
.. [3] https://tampub.uta.fi/bitstream/handle/10024/96864/GRADU-1428493916.pdf
 
.. [4] https://randu.org/tutorials/threads/
 
.. [5] https://www.gnu.org/gnu/linux-and-gnu.en.html

.. [6] http://blog.csdn.net/u010154760/article/details/45310513

.. [7] https://stackoverflow.com/questions/8639150/is-pthread-library-actually-a-user-thread-solution

.. [8] https://www.cnblogs.com/wangzahngjun/p/4977425.html (slab的简单解释)

.. [9] https://www.ibm.com/developerworks/cn/linux/l-cn-slub/ (slub的简单解释)

GNU
====


GNU Linux和Linux的区别和关系
================================

参考5 [5]_


user-thread/lwp
======================

内核中调度的单位是什么? 这首先是一个历史问题了.

POSIX, process, thread, light weight process(lwp), user-space-thread, green thread这些关系(历史), 参考6 [6]_

而在参考7 [7]_ 中, 提到关于内核的nptl的wiki:

  *NPTL is a so-called 1×1 threads library, in that threads created by the user (via the pthread_create() library function) are in 1-1 correspondence with schedulable entities in the kernel (tasks, in the Linux case). This is the simplest possible threading implementation.*
  
  --- nptl的维基

.. code-block:: python

   '''

   // pthread的man手册
   Linux implementations of POSIX threads
       Over time, two threading implementations have been provided by the
       GNU C library on Linux:

       LinuxThreads
              This is the original Pthreads implementation.  Since glibc
              2.4, this implementation is no longer supported.

       NPTL (Native POSIX Threads Library)
              This is the modern Pthreads implementation.  By comparison
              with LinuxThreads, NPTL provides closer conformance to the
              requirements of the POSIX.1 specification and better perfor‐
              mance when creating large numbers of threads.  NPTL is avail‐
              able since glibc 2.3.2, and requires features that are present
              in the Linux 2.6 kernel.

       Both of these are so-called 1:1 implementations, meaning that each
       thread maps to a kernel scheduling entity.  Both threading implemen‐
       tations employ the Linux clone(2) system call.  In NPTL, thread syn‐
       chronization primitives (mutexes, thread joining, and so on) are
       implemented using the Linux futex(2) system call.

   '''


glibc中nptl实现的pthread和内核中的task是1对1关系.

内核调度单位
===============

首先, linux一开始没有线程, 只有进程, 调度也是进程. 然后进程不方便, 需要线程(POSIX提出了线程的标准), 那linux内核一开始是没有线程这个东西的, 只有进程, 所以线程一开始就是用户态的概念.

然后内核在2.0版本开始去支持lwp, 也就是内核支持轻量级进程(lwp), 它和进程一样都是一个task结构, 不同的是, lwp的task结构包含了其他信息, 表示这个task是和其

父亲(也就是进程, 或者说线程组)共享一些资源的. 但是调度的时候, 内核依然是调度task结构, 只是会去判断task是否是lwp.

linux既调度进程, 也调度线程, 严格来说是调度task, 而进程和线程都映射到对应的task结构. 所以, 语义上, 内核调度是进程/线程/task都可以, 三者是同一个.

lwp, 进程, 线程可以通过ps命令来看:

.. code-block:: python

    '''
    
    thread.py启动一个线程. 然后ps -eLf | grep thread.py
    
    root 18234  9451 18234  2    2 17:35 ?        00:00:00 python3.6 thread_test.py
    root 18234  9451 18241  0    2 17:35 ?        00:00:00 python3.6 thread_test.py
    
    '''

可以看到, 线程和进程分别对应一个lwp, 然后进程的lwp和pid一致, 线程的pid和lwp是不一致的.

fork/clone
=============


KThread
===========

内线线程和lwp有区别是两个意思: lwp(task)是内核的调度单位, 内核线程也是对应一个task结构, 只是内核线程只能由内核去管理, 用户是终止不了的.

所以KThread被称为内核运行线程可能更好点

https://elixir.bootlin.com/linux/v4.15/source/include/linux/kthread.h


调度的周期/策略
================

下面的参考都出自参考3 [3]_, 基本上下面就是翻译了.

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

参考3 [3]_ 和参考4 [4]_


task结构参考: 





优先级
==========


.. code-block:: c

    int				prio;
    int				static_prio;
    int				normal_prio;
    unsigned int		rt_priority;
    const struct sched_class	*sched_class;
    struct sched_entity		se;


