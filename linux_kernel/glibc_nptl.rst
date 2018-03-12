glibc中的thread
=================

.. [1] https://www.gnu.org/gnu/linux-and-gnu.en.html

.. [2] http://blog.csdn.net/u010154760/article/details/45310513

.. [3] https://stackoverflow.com/questions/8639150/is-pthread-library-actually-a-user-thread-solution

.. [4] https://thorstenball.com/blog/2014/06/13/where-did-fork-go/

.. [5] https://lwn.net/Articles/534682/

.. [6] http://www.drdobbs.com/open-source/nptl-the-new-implementation-of-threads-f/184406204

.. [7] http://blog.csdn.net/hnwyllmm/article/details/45749063

.. [8] https://www.jianshu.com/p/ea692d4f5e27

参考6有Linux下Thread的详细介绍

参考7中对pthread的创建过程源码有一部分跟4.15的有区别, 注意一下

参考8有关于ARCH_CLONE的解释, 以及clone_flags等其他参数的一些解释

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

为什么glibc针对fork包装了一下呢. 先看看fork系统调用

.. code-block:: c

    #ifdef __ARCH_WANT_SYS_FORK
    SYSCALL_DEFINE0(fork)
    {
    #ifdef CONFIG_MMU
    	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
    #else
    	/* can not support in nommu mode */
    	return -EINVAL;
    #endif
    }
    #endif

fork系统调用基本上没有传参, 没什么灵活性.

而clone的系统调用:

.. code-block:: c

    #ifdef __ARCH_WANT_SYS_CLONE
    #ifdef CONFIG_CLONE_BACKWARDS
    SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
    		 int __user *, parent_tidptr,
    		 unsigned long, tls,
    		 int __user *, child_tidptr)
    #elif defined(CONFIG_CLONE_BACKWARDS2)
    SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
    		 int __user *, parent_tidptr,
    		 int __user *, child_tidptr,
    		 unsigned long, tls)
    #elif defined(CONFIG_CLONE_BACKWARDS3)
    SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
    		int, stack_size,
    		int __user *, parent_tidptr,
    		int __user *, child_tidptr,
    		unsigned long, tls)
    #else
    SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
    		 int __user *, parent_tidptr,
    		 int __user *, child_tidptr,
    		 unsigned long, tls)
    #endif
    {
        // ----------看这里, 这里才是一般性的定义!!!!!!
    	return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
    }
    #endif


不要被各种ifelse的宏定义给迷惑了, __ARCH_WANT_SYS_CLONE在X86架构下是定义了的, 然后忽略掉很多向后兼容的宏(CONFIG_CLONE_BACKWARDS2等等), 最后clone

也是调用_do_fork函数, 然后传参是不一样的, 并且有很多选项可以选, 灵活性更高.

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

所以, 我们使用glibc下的fork并不是系统调用fork, 而是glibc实现的一个wrap, 使用起来更容易, 并且内部是调用clone这个系统调用, 可以支持线程(lwp)的创建.

LinuxThread/nptl
===================

linux下POSIX线程的实现有两种: LinuxThread和nptl.

pthread的man手册有说明

.. code-block:: python

   '''

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

早期, LinuxThread并没有完全实现POSIX的标准, 并且使用了一个称为管理线程的角色去管理线程(参考 [3]_, 参考 [6]_).

由于LinuxThread这个库的一些缺点, 包括实现POSIX标准和性能, 后面被nptl给取代了, 直到现在.

  *It is instructive to understand the design choices that went into developing NPTL.*
  
  --- 参考6

关于nptl的实现, 又需要一些历史只知识了. nptl之前, ibm设计了m:n模型的NGPL, 然后linux社区讨论1:1和m:n的优劣势. 在O(1)的调度器被发布之后, 即使1:1下, 性能也不会那么糟糕.

  *After the release of NGPT, the Linux community debated the merits of M:N versus 1:1 threading models. When Ingo Molnar introduced the O(1) scheduler into the Linux kernel, however, the debate was largely closed.*
  
  *A 1:1 approach is simpler to implement, and with a constant time scheduler, there is no performance penalty*
  
  --- 参考6

nptl和clone, clone的改进是支持nptl的

  *In a 1:1 model, each thread has some characteristics of an entire process. Molnar, however, revised the clone() call to optimize thread creation. The kernel supports thread-specific data areas limited only by the available*
  
  --- 参考6

clone也让线程的创建更"便宜"(对比起LinuxThread), 当然初始化一个线程池总是一个好的实践

  *In short, using clone() to spawn a thread is no longer a heavyweight task. Application designers need no longer resort to thread pools created as part of the startup cost of an executable (although that may still be the correct design approach for certain applications).*
  
  --- 参考6

pthread结构
==============



pthread_create
===================

例如python中, 创建线程就直接调用pthread_create了


createthread
=====================

pthread_create会调用到createthread去实际创建线程

该函数一开始是在nptl/createthread.c中, 然后根据ChangeLog.18, 被移动到平台相关目录下sysdeps/unix/sysv/linux/createthread.c

该函数会调用clone, 但是是根据平台不同调用不同的clone的. 

.. code-block:: c

    static int
    create_thread (struct pthread *pd, const struct pthread_attr *attr,
    	       bool *stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
    {
    
    // 省略代码
    
    // 这里设置了clone的flag
    const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
          		   | CLONE_SIGHAND | CLONE_THREAD
          		   | CLONE_SETTLS | CLONE_PARENT_SETTID
          		   | CLONE_CHILD_CLEARTID
          		   | 0);
    
    TLS_DEFINE_INIT_TP (tp, pd);
    
    // 调用平台相关的clone
    if (__glibc_unlikely (ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
          			    clone_flags, pd, &pd->tid, tp, &pd->tid)
          		== -1))
      return errno;
    
    // 省略代码
    
    }


关于ARCH_CLONE这个宏

  *这里 ARCH_CLONE 是 glibc 对底层做的一层封装，它是直接使用的 ABI 接口，代码是用汇编语言写的，x86_64 平台的代码在 (sysdeps/unix/sysv/linux/x86_64/clone.S) 文件中， 感兴趣可以自己去看。你会发现其实就是就是调用了 linux 提供的 clone 接口。所以也可以直接参考 Linux 手册上对 clone 函数的描述，此宏与 clone 参数是一样的。 我们可以看出此处，函数两次传入的都子线程 pthread 中 tid 值，以让内核在线程开始时设置线程 ID 以及线程结束时清除其 ID 值。这样此线程的栈内存块就可以被随后的线程释放了。*
  
  -- 参考8

关于各种flag, 注释上有

.. code-block:: c

    /*
    
         CLONE_VM, CLONE_FS, CLONE_FILES
    	These flags select semantics with shared address space and
    	file descriptors according to what POSIX requires.
    
         CLONE_SIGHAND, CLONE_THREAD
    	This flag selects the POSIX signal semantics and various
    	other kinds of sharing (itimers, POSIX timers, etc.).
    
         CLONE_SETTLS
    	The sixth parameter to CLONE determines the TLS area for the
    	new thread.
    
         CLONE_PARENT_SETTID
    	The kernels writes the thread ID of the newly created thread
    	into the location pointed to by the fifth parameters to CLONE.
    
    	Note that it would be semantically equivalent to use
    	CLONE_CHILD_SETTID but it is be more expensive in the kernel.
    
         CLONE_CHILD_CLEARTID
    	The kernels clears the thread ID of a thread that has called
    	sys_exit() in the location pointed to by the seventh parameter
    	to CLONE.
    */


参考 [8]_有比较多的解释



