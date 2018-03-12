glibc中的thread
=================

.. [1] https://www.gnu.org/gnu/linux-and-gnu.en.html

.. [2] http://blog.csdn.net/u010154760/article/details/45310513

.. [3] https://stackoverflow.com/questions/8639150/is-pthread-library-actually-a-user-thread-solution

.. [4] https://thorstenball.com/blog/2014/06/13/where-did-fork-go/

.. [5] https://lwn.net/Articles/534682/

.. [6] http://www.drdobbs.com/open-source/nptl-the-new-implementation-of-threads-f/184406204 (nptl的介绍)


GNU
====

GNU这个组织


GNU Linux和Linux的区别和关系
================================

参考 [1]_


thread/lwp
======================

内核中调度的单位是什么? 这首先是一个历史问题了.

POSIX, process, thread, light weight process(lwp), user-space-thread, green thread这些关系(历史), 参考 [2]_

而在参考 [3]_ 中, 提到关于内核的nptl的wiki:

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

fork/clone调用
================

fork/clone会在线程创建的时候被调用, 先来个了解.

当我们调用fork的时候, 并不会直接调用fork这个系统调用, 而是调用相关库的fork函数, 比如glibc的fork.

关于glibc的fork/clone, 以及内核的fork调用:

  *Since  version  2.3.3,  rather than invoking the kernel's fork() system call, the glibc fork() wrapper that is provided as part of the NPTL threading implementation invokes clone(2) with flags that
  provide the same effect as the traditional system call.  (A call to fork() is equivalent to a call to clone(2) specifying flags as just SIGCHLD.)  The glibc wrapper invokes any fork  handlers  that
  have been established using pthread_atfork(3).*
  
  --- fork的man手册

为什么glibc针对fork包装了一下呢:

  *After digging around a bit(https://lwn.net/Articles/534682/) I found out that making a system call is actually harder than just calling fork() somewhere in my code. I’d need to know the unique number of system call I was about to make, set up registers, call a special instruction (which varies on different machine architectures) to switch to kernel mode and then handle the results when I’m back in user space.
  
  By providing a wrapper around certain system calls glibc makes it a lot easier and portable for developers to use system calls. There is still the possibility to use syscall(2) to call system calls somewhat more directly.*
  
  --- 参考4

所以, glibc下的fork是不会去调用fork系统调用, 而是自己实现了一层wrap. 这是因为直接调用fork系统调用的话, 需要自己设置

寄存器什么的, 很麻烦(系统调用总是赤裸裸的), 而做一层wrap之后, 开发者使用fork就更容易(c库会帮你设置寄存器什么的), 并且fork更portable, 并且

fork调用的是clone而不是原生的fork调用, 这是因为clone支持新建一个线程(lwp).

所在在内核看来, 没有线程和进程的区别, 只有进程, 区别在于一个进程是否和其他进程共享数据, 如果共享了, 就是lwp, 也就是线程.

为什么glibc的fork针对fork调用做了wrap之后, 调用的是clone而不是fork?

  *In contrast to fork(2), which takes no arguments, we can call clone(2) with different arguments to change which process will be created. Do they need to share their execution context? Memory? File descriptors? Signal handlers? clone(2) allows us to change these attributes of newly created processes. This is clearly much more flexible and powerful than fork(2), which creates the “fat processes” we can see when we run ps.*
  
  --- 参考4

也就是clone更灵活, 并且可以创建线程线程.

  *In contrast to fork(2), which takes no arguments, we can call clone(2) with different arguments to change which process will be created*
  
  --- 参考4

pthread结构
==============



pthread_create
===================




