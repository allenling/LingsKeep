Linux的pthread(nptl)
======================

.. [1] https://www.gnu.org/gnu/linux-and-gnu.en.html

.. [2] http://blog.csdn.net/u010154760/article/details/45310513

.. [3] https://stackoverflow.com/questions/8639150/is-pthread-library-actually-a-user-thread-solution

.. [4] https://thorstenball.com/blog/2014/06/13/where-did-fork-go/

.. [5] https://lwn.net/Articles/534682/

.. [6] http://www.drdobbs.com/open-source/nptl-the-new-implementation-of-threads-f/184406204

.. [7] http://blog.csdn.net/hnwyllmm/article/details/45749063

.. [8] https://www.jianshu.com/p/ea692d4f5e27

.. [9] https://casatwy.com/pthreadde-ge-chong-tong-bu-ji-zhi.html

.. [10] https://stackoverflow.com/questions/18904292/is-it-true-that-fork-calls-clone-internally

.. [11] http://kernel.meizu.com/linux-signal.html

.. [12] http://www.cnblogs.com/parrynee/archive/2010/01/14/1648152.html

.. [13] http://cs-pub.bu.edu/fac/richwest/cs591_w1/notes/wk3_pt2.PDF

参考4是fork的一些解释

参考6有Linux下Thread的历史介绍

参考7中对pthread的创建过程源码有一部分跟4.15的有区别, 注意一下

参考8有关于ARCH_CLONE的解释, 以及clone_flags等其他参数的一些解释

参考9是pthread下的同步机制

参考11则是信号处理(kill等)的代码流程

参考12是task结构中, pids这个属性的一些解释. 注意的是4.15类型变成了pid_link而不是文章中的pid_type, 但是都是使用hash表结构.

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

而glibc中的fork怎么实现的? 

sysdeps/nptl/fork.c

.. code-block:: c

    pid_t
    __libc_fork (void)
    {
    
    // 省略代码
    
    // 这里调用平台相关的fork
    #ifdef ARCH_FORK
      pid = ARCH_FORK ();
    #else
    # error "ARCH_FORK must be defined so that the CLONE_SETTID flag is used"
      pid = INLINE_SYSCALL (fork, 0);
    #endif
    
    // 省略代码
    
    }

然后在linux x86_64平台下, ARCH_FORK有

sysdeps/unix/sysv/linux/x86_64/arch-fork.h

.. code-block:: c

    #define ARCH_FORK() \
      INLINE_SYSCALL (clone, 4,                                                   \
                      CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD, 0,     \
                      NULL, &THREAD_SELF->tid)

linux(x86_64)下fork是去调用clone, 传入的clone_flag主要区别是SIGCHLD

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

----

task和thread
=================

下面从信号处理流程去看task中的结构信息的作用, 这里不涉及调度, 调度参考linux_task_schedule.rst

    *理解信号异步机制的关键是信号的响应时机，我们对一个进程发送一个信号以后，其实并没有硬中断发生，只是简单把信号挂载到目标进程的信号 pending 队列上去，信号真正得到执行的时机是进程执行完异常/中断返回到用户态的时刻。
    
    让信号看起来是一个异步中断的关键就是，正常的用户进程是会频繁的在用户态和内核态之间切换的（这种切换包括：系统调用、缺页异常、系统中断…），所以信号能很快的能得到执行。但这也带来了一点问题，内核进程是不响应信号的，除非它刻意的去查询。所以通常情况下我们无法通过kill命令去杀死一个内核进程。*
    
    --- 参考11

下面信号处理的代码参考 [11]_


kill发送信号
================


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L2936

.. code-block:: c

    /**
     *  sys_kill - send a signal to a process
     *  @pid: the PID of the process
     *  @sig: signal to be sent
     */
    SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
    {
        struct siginfo info;

        info.si_signo = sig;
        info.si_errno = 0;
        info.si_code = SI_USER;
        info.si_pid = task_tgid_vnr(current);
        info.si_uid = from_kuid_munged(current_user_ns(), current_uid());

        return kill_something_info(sig, &info, pid);
    }

这里传入的pid是pid_t类型, 而这个pid_t的定义是在

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/types.h#L22
    typedef __kernel_pid_t		pid_t;


然后搜索一下, 看到似乎这个__kernel_pid_t是跟平台有关的, 没找到x86_64的, 就看到什么安腾(ia)的, 所以

只能以在posix_types下的定义为准了, 是一个int类型

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/uapi/asm-generic/posix_types.h#L28
    #ifndef __kernel_pid_t
    typedef int		__kernel_pid_t;
    #endif

kill_something_info
======================

https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1399

.. code-block:: c

    /*
     * kill_something_info() interprets pid in interesting ways just like kill(2).
     *
     * POSIX specifies that kill(-1,sig) is unspecified, but what we have
     * is probably wrong.  Should make it like BSD or SYSV.
     */
    
    static int kill_something_info(int sig, struct siginfo *info, pid_t pid)
    {
    	int ret;
    
        // 如果pid大于0, 那么会发送到对应的进程中
    	if (pid > 0) {
    		rcu_read_lock();
    		ret = kill_pid_info(sig, info, find_vpid(pid));
    		rcu_read_unlock();
    		return ret;
    	}
    
    	/* -INT_MIN is undefined.  Exclude this case to avoid a UBSAN warning */
    	if (pid == INT_MIN)
    		return -ESRCH;
    
    	read_lock(&tasklist_lock);
        // (pid <= 0) && (pid != -1), 发送信号给pid进程所在进程组中的每一个线程组
    	if (pid != -1) {
    		ret = __kill_pgrp_info(sig, info,
    				pid ? find_vpid(-pid) : task_pgrp(current));
    	} else {
                // pid = -1, 发送信号给所有进程的进程组，除了pid=1和当前进程自己
    		int retval = 0, count = 0;
    		struct task_struct * p;
    
    		for_each_process(p) {
    			if (task_pid_vnr(p) > 1 &&
    					!same_thread_group(p, current)) {
    				int err = group_send_sig_info(sig, info, p);
    				++count;
    				if (err != -EPERM)
    					retval = err;
    			}
    		}
    		ret = count ? retval : -ESRCH;
    	}
    	read_unlock(&tasklist_lock);
    
    	return ret;
    }

kill_pid_info
==================


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1313

.. code-block:: c

    int kill_pid_info(int sig, struct siginfo *info, struct pid *pid)
    {
    	int error = -ESRCH;
    	struct task_struct *p;
    
    	for (;;) {
    		rcu_read_lock();
    		p = pid_task(pid, PIDTYPE_PID);
                // 这里通过pid获取对应的task结构
    		if (p)
                        // 把信号发送到进程
                        // 也就是把信号发送到线程组
    			error = group_send_sig_info(sig, info, p);
    		rcu_read_unlock();
    		if (likely(!p || error != -ESRCH))
    			return error;
    
    		/*
    		 * The task was unhashed in between, try again.  If it
    		 * is dead, pid_task() will return NULL, if we race with
    		 * de_thread() it will find the new leader.
    		 */
    	}
    }


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1279

.. code-block:: c

    int group_send_sig_info(int sig, struct siginfo *info, struct task_struct *p)
    {
    	int ret;
    
    	rcu_read_lock();
    	ret = check_kill_permission(sig, info, p);
    	rcu_read_unlock();
    
    	if (!ret && sig)
                // 最后还是调用do_send_sig_info
                // !!!!!注意, 这里最后一个参数是true!!!
    		ret = do_send_sig_info(sig, info, p, true);
    
    	return ret;
    }

https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1155

.. code-block:: c

    int do_send_sig_info(int sig, struct siginfo *info, struct task_struct *p,
    			bool group)
    {
    	unsigned long flags;
    	int ret = -ESRCH;
    
    	if (lock_task_sighand(p, &flags)) {
                // !!!!这里, 上面最后一个参数是group, 传参的时候传的是true!!!
    		ret = send_signal(sig, info, p, group);
    		unlock_task_sighand(p, &flags);
    	}
    
    	return ret;
    }


__send_signal
================

上面的do_send_sig_info->send_signal最后会调用到__send_signal


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L994


.. code-block:: c

    static int __send_signal(int sig, struct siginfo *info, struct task_struct *t,
    			int group, int from_ancestor_ns)
    {
    
    
    	struct sigpending *pending;
    	struct sigqueue *q;
    	int override_rlimit;
    	int ret = 0, result;
    
    	assert_spin_locked(&t->sighand->siglock);
    
    	result = TRACE_SIGNAL_IGNORED;
        // !!!判断是否可以忽略信号
    	if (!prepare_signal(sig, t,
    			from_ancestor_ns || (info == SEND_SIG_FORCED)))
    		goto ret;

        // !!注意这里, 这里如果group是true的话
        // 那么pending是t->signal->shared_pendding, 说明是拿线程组中共享的信号队列
        // 如果group不是true, 那么拿的是task自己的pending
    	pending = group ? &t->signal->shared_pending : &t->pending;

        /*
         * Short-circuit ignored signals and support queuing
         * exactly one non-rt signal, so that we can get more
         * detailed information about the cause of the signal.
         */
        result = TRACE_SIGNAL_ALREADY_PENDING;

        // 这里legacy_queue判断, 如果sig是常规信号, 那么是否已经在队列中了, 如果在了就过
        // 如果sig是实时信号, 则可以重复入队
        // 另外一方面也说明了，如果是实时信号，尽管信号重复，但还是要加入pending队列
        // 实时信号的多个信号都需要能被接收到
        if (legacy_queue(pending, sig))
        	goto ret;
        
        result = TRACE_SIGNAL_DELIVERED;
        /*
         * fast-pathed signals for kernel-internal things like SIGSTOP
         * or SIGKILL.
         */
        // 如果是一些强制信号, 那么直接处理
        // 如果是强制信号(SEND_SIG_FORCED)，不走挂载pending队列的流程，直接快速路径优先处理
        if (info == SEND_SIG_FORCED)
            goto out_set;    
        
        /*
         * Real-time signals must be queued if sent by sigqueue, or
         * some other real-time mechanism.  It is implementation
         * defined whether kill() does so.  We attempt to do so, on
         * the principle of least surprise, but since kill is not
         * allowed to fail with EAGAIN when low on memory we just
         * make sure at least one signal gets delivered and don't
         * pass on the info struct.
         */

        // 符合条件的特殊信号可以突破siganl pending队列的大小限制(rlimit)
        // 否则在队列满的情况下，丢弃信号
        // signal pending队列大小rlimit的值可以通过命令"ulimit -i"查看
        if (sig < SIGRTMIN)
        	override_rlimit = (is_si_special(info) || info->si_code >= 0);
        else
        	override_rlimit = 0;
        
        // 没有ignore的信号，加入到pending队列中
        // pending队列的每一个元素都是sigqueue结构
        q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);

        // 加入pending队列
        if (q) {
        	list_add_tail(&q->list, &pending->list);
        	switch ((unsigned long) info) {
        	case (unsigned long) SEND_SIG_NOINFO:
        		q->info.si_signo = sig;
        		q->info.si_errno = 0;
        		q->info.si_code = SI_USER;
        		q->info.si_pid = task_tgid_nr_ns(current,
        						task_active_pid_ns(t));
        		q->info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
        		break;
        	case (unsigned long) SEND_SIG_PRIV:
        		q->info.si_signo = sig;
        		q->info.si_errno = 0;
        		q->info.si_code = SI_KERNEL;
        		q->info.si_pid = 0;
        		q->info.si_uid = 0;
        		break;
        	default:
        		copy_siginfo(&q->info, info);
        		if (from_ancestor_ns)
        			q->info.si_pid = 0;
        		break;
        	}
        
        	userns_fixup_signal_uid(&q->info, t);
        
        } else if (!is_si_special(info)) {
        	if (sig >= SIGRTMIN && info->si_code != SI_USER) {
        		/*
        		 * Queue overflow, abort.  We may abort if the
        		 * signal was rt and sent by user using something
        		 * other than kill().
        		 */
        		result = TRACE_SIGNAL_OVERFLOW_FAIL;
        		ret = -EAGAIN;
        		goto ret;
        	} else {
        		/*
        		 * This is a silent loss of information.  We still
        		 * send the signal, but the *info bits are lost.
        		 */
        		result = TRACE_SIGNAL_LOSE_INFO;
        	}
        }
    
    

        out_set:
        	signalfd_notify(t, sig);
        	sigaddset(&pending->signal, sig);
                // 选择合适的进程来响应信号，如果需要并唤醒对应的进程
        	complete_signal(sig, t, group);
        ret:
        	trace_signal_generate(sig, info, t, group, result);
        	return ret;
            
    }

complete_signal
==================

这里会选择合适的task去唤醒, 调用wants_signal去检查task是否可以处理信号

.. code-block:: c

    static void complete_signal(int sig, struct task_struct *p, int group)
    {
    	struct signal_struct *signal = p->signal;
    	struct task_struct *t;
    
    	/*
    	 * Now find a thread we can wake up to take the signal off the queue.
    	 *
    	 * If the main thread wants the signal, it gets first crack.
    	 * Probably the least surprising to the average bear.
    	 */
        // 注释上说, 先检查主线程是否可以处理信号
        // 如果可以, 主线程处理
    	if (wants_signal(sig, p))
    		t = p;
    	else if (!group || thread_group_empty(p))
    		/*
    		 * There is just one thread and it does not need to be woken.
    		 * It will dequeue unblocked signals before it runs again.
    		 */
    		return;
    	else {
    		/*
    		 * Otherwise try to find a suitable thread.
    		 */
    		t = signal->curr_target;
                // 否则一个一个去遍历线程, 直到找到一个
                // 线程可以处理信号
    		while (!wants_signal(sig, t)) {
    			t = next_thread(t);
    			if (t == signal->curr_target)
    				/*
    				 * No thread needs to be woken.
    				 * Any eligible threads will see
    				 * the signal in the queue soon.
    				 */
    				return;
    		}
    		signal->curr_target = t;
    	}
    
    	/*
    	 * Found a killable thread.  If the signal will be fatal,
    	 * then start taking the whole group down immediately.
    	 */
        // 注释上说, 如果信号是一些致命的信号
        // 那么遍历所有的task, 每个task的pending队列设置上SIGKILL标志位
        // 然后唤醒task, 也就是杀死task
        if (sig_fatal(p, sig) &&
    	    !(signal->flags & SIGNAL_GROUP_EXIT) &&
    	    !sigismember(&t->real_blocked, sig) &&
    	    (sig == SIGKILL || !p->ptrace)) {
    		/*
    		 * This signal will be fatal to the whole group.
    		 */
    		if (!sig_kernel_coredump(sig)) {
    			/*
    			 * Start a group exit and wake everybody up.
    			 * This way we don't have other threads
    			 * running and doing things after a slower
    			 * thread has the fatal signal pending.
    			 */
    			signal->flags = SIGNAL_GROUP_EXIT;
    			signal->group_exit_code = sig;
    			signal->group_stop_count = 0;
    			t = p;
    			do {
                                // 逐个杀死task
    				task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK);
    				sigaddset(&t->pending.signal, SIGKILL);
    				signal_wake_up(t, 1);
    			} while_each_thread(p, t);
    			return;
    		}
    	}
    
    	/*
    	 * The signal is already in the shared-pending queue.
    	 * Tell the chosen thread to wake up and dequeue it.
    	 */
        // 唤醒task
    	signal_wake_up(t, sig == SIGKILL);
    	return;
    }

next_thread
===============

获取task中线程组中的下一个线程

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L558

.. code-block:: c

    static inline struct task_struct *next_thread(const struct task_struct *p)
    {
    	return list_entry_rcu(p->thread_group.next,
    			      struct task_struct, thread_group);
    }

下一个线程就是thread_group.next了, 所以可以推测线程都是通过thread_group连接起来的

wants_signal
==============

判断线程是否可以处理进程


.. code-block:: c

    /*
     * Test if P wants to take SIG.  After we've checked all threads with this,
     * it's equivalent to finding no threads not blocking SIG.  Any threads not
     * blocking SIG were ruled out because they are not running and already
     * have pending signals.  Such threads will dequeue from the shared queue
     * as soon as they're available, so putting the signal on the shared queue
     * will be equivalent to sending it to one such thread.
     */
    static inline int wants_signal(int sig, struct task_struct *p)
    {
        if (sigismember(&p->blocked, sig))
            return 0;
        if (p->flags & PF_EXITING)
            return 0;
        if (sig == SIGKILL)
            return 1;
        if (task_is_stopped_or_traced(p))
            return 0;
        return task_curr(p) || !signal_pending(p);
    }

1. sigismember作用是: *test wehether signum is a member of set.(&p->blocked, sig)* , 也就是是否线程是否block了信号.
   因为线程可以调用sigprocmask/pthread_sigmask去block指定的信号, 如果结果为真, 表示线程屏蔽了信号.
   可以参考 `这里 <http://devarea.com/linux-handling-signals-in-a-multithreaded-application/#.WpAhGINuaUk>`_
   
2. PF_EXITING表示进程退出状态

3. SIGKILL这个信号是要传递给所有的线程的(这样才能达到kill的目的), 所以返回1

4. task_is_stopped_or_traced线程是否是终止状态

5. task_curr是判断当前线程是否占用cpu, *task_curr - is this task currently executing on a CPU?*

signal_pending
================

先看看函数调用过程

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L313
    static inline int signal_pending(struct task_struct *p)
    {
    	return unlikely(test_tsk_thread_flag(p,TIF_SIGPENDING));
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1536
    static inline int test_tsk_thread_flag(struct task_struct *tsk, int flag)
    {
        // 这里调用task_thread_info去拿task结构的thread_info
    	return test_ti_thread_flag(task_thread_info(tsk), flag);
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/thread_info.h#L77
    static inline int test_ti_thread_flag(struct thread_info *ti, int flag)
    {
    	return test_bit(flag, (unsigned long *)&ti->flags);
    }

而task_thread_info函数则是一般去拿task结构的thread_info

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1456

.. code-block:: c

    #ifdef CONFIG_THREAD_INFO_IN_TASK
    static inline struct thread_info *task_thread_info(struct task_struct *task)
    {
    	return &task->thread_info;
    }
    #elif !defined(__HAVE_THREAD_FUNCTIONS)
    # define task_thread_info(task)	((struct thread_info *)(task)->stack)
    #endif

所以, signal_pending则是去寻找task对应的thread_info是否有设置上了TIF_SIGPENDING标志位


唤醒的是哪个线程?
===================

经过测试, 无论主线程是一直占着cpu还是陷入等待(sleep), signal一般唤醒的都是主线程. 下面是测试源码

.. code-block:: c

    #include<stdio.h>
    #include<unistd.h>
    #include<pthread.h>
    #include <sys/mman.h>
    #include <stdlib.h>
    #include <sys/prctl.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <sys/ioctl.h>
     
    void *threadfn1(void *p)
    {
    	while(1){
    		printf("thread1\n");
    		sleep(2);
    	}
    	return 0;
    }
     
    void *threadfn2(void *p)
    {
        pthread_t   tid;
        tid = pthread_self();
    	while(1){
    		printf("thread2: %ld\n", (long) tid);
    		sleep(2);
    	}
    	return 0;
    }
     
    void *threadfn3(void *p)
    {
        pthread_t   tid;
        tid = pthread_self();
    	while(1){
    		printf("thread3: %ld\n", (long) tid);
    		sleep(2);
    	}
    	return 0;
    }
     
     
    void handler(int signo, siginfo_t *info, void *extra) 
    {
    	int i;
        pthread_t   tid;
        tid = pthread_self();
    	for(i=0;i<10;i++)
    	{
    		puts("signal");
            printf("in %ld\n", (long) tid);
    		sleep(2);
    	}
    }
     
    void set_sig_handler(void)
    {
            struct sigaction action;
     
     
            action.sa_flags = SA_SIGINFO; 
            action.sa_sigaction = handler;
    
            if (sigaction(SIGRTMIN + 3, &action, NULL) == -1) { 
                perror("sigusr: sigaction");
                _exit(1);
            }
     
    }
     
    int main()
    {
    	pthread_t t1,t2,t3;
        pthread_t   tid;
        tid = pthread_self();
        printf("main thread: %ld\n", (long)tid);
    	set_sig_handler();
    	// pthread_create(&t1,NULL,threadfn1,NULL);
    	pthread_create(&t2,NULL,threadfn2,NULL);
    	pthread_create(&t3,NULL,threadfn3,NULL);
        int count = 0;
        // sleep(3600);
        // 下面的while可以换成sleep
        while (1){
            count += 1;
        }
    	pthread_exit(NULL);
    	return 0;
    }

在main中

1. 无论是while 1计算, 还是sleep, 发送signal(*sudo kill -s 37 pid*)之后总是唤醒的总是主线程

2. 只开启一个子线程, 比如子线程2, 然后子线程2中密集计算(while count += 1), 然后主线程sleep, 依然是唤醒主线程.

**所以, 也就是对主线程调用wants_signal之后, 总是ture.**

所以, complete_signal->signal_wake_up->signal_wake_up_state会发中断, 让线程去执行信号处理, 如果线程正在计算, 也会处理这个中断的.

根据参考 [13]_的一些解释:

*CPU checks for interrupts after executing each instruction.*

cpu每一执行一个指令之后, 都会去检查中断

*If interrupt occurred, control unit: Determines vector i, corresponding to interrupt, (省略一些步骤), If necessary, switches to new stack by

Loading ss & esp regs with values found in the task state segment (TSS) of current process, (省略一些步骤), Interrupt handler is then executed!*

简单来说就是拿到signal handler的栈什么的和参数, 然后执行.

根据参考 [12]_中的解释, 会保存当前执行函数的栈信息什么的, 切换到用户态执行signal handler, 然后回到内核, 然后再执行之前保存的函数.


block信号
=============

可以使用sigprocmask/pthread_sigmask去block指定的信号, 前者是线程组, 后者是指定的线程.


signal_wake_up
=================

唤醒task


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L349
    static inline void signal_wake_up(struct task_struct *t, bool resume)
    {
    	signal_wake_up_state(t, resume ? TASK_WAKEKILL : 0);
    }


    // https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L661
    /*
     * Tell a process that it has a new active signal..
     *
     * NOTE! we rely on the previous spin_lock to
     * lock interrupts for us! We can only be called with
     * "siglock" held, and the local interrupt must
     * have been disabled when that got acquired!
     *
     * No need to set need_resched since signal event passing
     * goes through ->blocked
     */
    void signal_wake_up_state(struct task_struct *t, unsigned int state)
    {
        // 这里设置task的thread_info的flag是TIF_SIGPENDING
    	set_tsk_thread_flag(t, TIF_SIGPENDING);
    	/*
    	 * TASK_WAKEKILL also means wake it up in the stopped/traced/killable
    	 * case. We don't check t->state here because there is a race with it
    	 * executing another processor and just now entering stopped state.
    	 * By using wake_up_state, we ensure the process will wake up and
    	 * handle its death signal.
    	 */
        // wake_up_state则是去唤醒task!!!!
    	if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
    		kick_process(t);
    }



sigaction
============

  *The original Linux system call was named sigaction().  However, with the addition of real-time signals in Linux 2.2, the fixed-size, 32-bit sigset_t type supported by that system call was no longer
  fit  for  purpose.  Consequently, a new system call, rt_sigaction(), was added to support an enlarged sigset_t type.  The new system call takes a fourth argument, size_t sigsetsize, which specifies
  the size in bytes of the signal sets in act.sa_mask and oldact.sa_mask.  This argument is currently required to have the value sizeof(sigset_t) (or the error EINVAL results).  The glibc sigaction()
  wrapper function hides these details from us, transparently calling rt_sigaction() when the kernel provides it.*
  
  --- sigaction的man手册

根据man手册上的说明, rt_sigaction这个系统调用是取代旧的sigaction系统调用, 并且glibc中的sigaction函数将会调用rt_sigaction这个系统调用

所以, 我们调用sigaction的时候, 其实是调用glibc的sigaction, glibc对一些系统调用进行了wrap, 比如fork和clone.


linux的x86_64架构下的sigaction

sysdeps/unix/sysv/linux/x86_64/sigaction.c


.. code-block:: c

    int
    __libc_sigaction (int sig, const struct sigaction *act, struct sigaction *oact)
    {
      int result;
      struct kernel_sigaction kact, koact;
    
      if (act)
        {
          kact.k_sa_handler = act->sa_handler;
          memcpy (&kact.sa_mask, &act->sa_mask, sizeof (sigset_t));
          kact.sa_flags = act->sa_flags | SA_RESTORER;
    
          kact.sa_restorer = &restore_rt;
        }
    
      /* XXX The size argument hopefully will have to be changed to the
         real size of the user-level sigset_t.  */
      // 这里!!!调用了系统调用rt_sigaction
      result = INLINE_SYSCALL (rt_sigaction, 4,
    			   sig, act ? &kact : NULL,
    			   oact ? &koact : NULL, _NSIG / 8);
      if (oact && result >= 0)
        {
          oact->sa_handler = koact.k_sa_handler;
          memcpy (&oact->sa_mask, &koact.sa_mask, sizeof (sigset_t));
          oact->sa_flags = koact.sa_flags;
          oact->sa_restorer = koact.sa_restorer;
        }
      return result;
    }


glibc的sigaction函数只是帮我们组装了sigaction结构, 然后调用rt_sigaction系统调用.

而rt_sigaction的系统调用是在https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c找到, 其中会根据宏定义的不同去有不同的实现.

但是本质上, 最终调用的还是do_sigaction这个函数


do_sigaction
================

这个函数的作用是把current, 也就是当前task, 的信号处理函数替换成用户指定的函数


.. code-block:: c

    int do_sigaction(int sig, struct k_sigaction *act, struct k_sigaction *oact)
    {
    	struct task_struct *p = current, *t;
    	struct k_sigaction *k;
    	sigset_t mask;
    
    	if (!valid_signal(sig) || sig < 1 || (act && sig_kernel_only(sig)))
    		return -EINVAL;
    
        // !!!拿到当前task的信号处理函数!!!!!
    	k = &p->sighand->action[sig-1];
    
    	spin_lock_irq(&p->sighand->siglock);
    	if (oact)
    		*oact = *k;
    
    	sigaction_compat_abi(act, oact);
    
    	if (act) {
    		sigdelsetmask(&act->sa.sa_mask,
    			      sigmask(SIGKILL) | sigmask(SIGSTOP));
                // !!!这里替换掉用户指定的信号函数
    		*k = *act;
    		/*
    		 * POSIX 3.3.1.3:
    		 *  "Setting a signal action to SIG_IGN for a signal that is
    		 *   pending shall cause the pending signal to be discarded,
    		 *   whether or not it is blocked."
    		 *
    		 *  "Setting a signal action to SIG_DFL for a signal that is
    		 *   pending and whose default action is to ignore the signal
    		 *   (for example, SIGCHLD), shall cause the pending signal to
    		 *   be discarded, whether or not it is blocked"
    		 */
                // 下面这个判断是该信号是否被ignore
                // sig_handler这个拿到sig的handler, 如果handler是SIG_IGN
                // 那么表示忽略
                // 忽略的时候把所有线程的中的该signale从pending移除
    		if (sig_handler_ignored(sig_handler(p, sig), sig)) {
    			sigemptyset(&mask);
    			sigaddset(&mask, sig);
    			flush_sigqueue_mask(&mask, &p->signal->shared_pending);
    			for_each_thread(p, t)
    				flush_sigqueue_mask(&mask, &t->pending);
    		}
    	}
    
    	spin_unlock_irq(&p->sighand->siglock);
    	return 0;
    }


flush_sigqueue_mask的注释是: Remove signals in mask from the pending set and queue.

----

task和线程部分
================


通过上面的信号处理流程知道, 在创建线程的时候, 有几个关键的属性




task结构
============

pthread到task的关键代码, 其实就是clone系统调用新建task.

https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L2132

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
        // 看这里!!!!!!!!!!!!!!!
    	return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
    }
    #endif

clone也会调用_do_fork, 根据上一节, 传入了很多clone_flags, 其中有CLONE_THREAD, 然后_do_fork有

https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L2015

.. code-block:: c


    long _do_fork(unsigned long clone_flags,
    	      unsigned long stack_start,
    	      unsigned long stack_size,
    	      int __user *parent_tidptr,
    	      int __user *child_tidptr,
    	      unsigned long tls)
    {
        // 一个新的task结构
    	struct task_struct *p;
    	int trace = 0;
    	long nr;
    
    	/*
    	 * Determine whether and which event to report to ptracer.  When
    	 * called from kernel_thread or CLONE_UNTRACED is explicitly
    	 * requested, no event is reported; otherwise, report if the event
    	 * for the type of forking is enabled.
    	 */
        // 这里暂时看不懂
    	if (!(clone_flags & CLONE_UNTRACED)) {
    		if (clone_flags & CLONE_VFORK)
    			trace = PTRACE_EVENT_VFORK;
    		else if ((clone_flags & CSIGNAL) != SIGCHLD)
    			trace = PTRACE_EVENT_CLONE;
    		else
    			trace = PTRACE_EVENT_FORK;
    
    		if (likely(!ptrace_event_enabled(current, trace)))
    			trace = 0;
    	}
    
        // --------注意, 这里我们复制task了!!!!
        p = copy_process(clone_flags, stack_start, stack_size,
    			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    	add_latent_entropy();
    	/*
    	 * Do this prior waking up the new thread - the thread pointer
    	 * might get invalid after that point, if the thread exits quickly.
    	 */
    	if (!IS_ERR(p)) {
    		struct completion vfork;
    		struct pid *pid;
    
    		trace_sched_process_fork(current, p);
    
    		pid = get_task_pid(p, PIDTYPE_PID);
    		nr = pid_vnr(pid);
    
    		if (clone_flags & CLONE_PARENT_SETTID)
    			put_user(nr, parent_tidptr);
    
    		if (clone_flags & CLONE_VFORK) {
    			p->vfork_done = &vfork;
    			init_completion(&vfork);
    			get_task_struct(p);
    		}
    
                // 没有错误, 我们就启动task了
    		wake_up_new_task(p);
    
    		/* forking complete and child started to run, tell ptracer */
    		if (unlikely(trace))
    			ptrace_event_pid(trace, pid);
    
    		if (clone_flags & CLONE_VFORK) {
    			if (!wait_for_vfork_done(p, &vfork))
    				ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
    		}
    
    		put_pid(pid);
    	} else {
    		nr = PTR_ERR(p);
    	}
    	return nr;
    }

copy_process
===============

这里是复制的操作, 太长, 先暂时省略很多很多很多代码

https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L1534

.. code-block:: c

    /*
     * This creates a new process as a copy of the old one,
     * but does not actually start it yet.
     *
     * It copies the registers, and all the appropriate
     * parts of the process environment (as per the clone
     * flags). The actual kick-off is left to the caller.
     */
    static __latent_entropy struct task_struct *copy_process(
    					unsigned long clone_flags,
    					unsigned long stack_start,
    					unsigned long stack_size,
    					int __user *child_tidptr,
    					struct pid *pid,
    					int trace,
    					unsigned long tls,
    					int node)
    {
    
        // 省略代码
        
        // 你看, 复制task结构了
        p = dup_task_struct(current, node);
        
        // 省略代码

        // 这里会根据是否是线程去决定是否公用
        // 信号结构
        retval = copy_signal(clone_flags, p);
        if (retval)
        	goto bad_fork_cleanup_sighand;

        // 省略代码

        // 这里的pid则是task结构的pid
        // 和我们通常称的pid是不太一样
        p->pid = pid_nr(pid);

        // 下面是针对线程, 赋值task结构里面的属性
        // 包括什么tgid呀
        if (clone_flags & CLONE_THREAD) {
        	p->exit_signal = -1;
        	p->group_leader = current->group_leader;
                // 如果是线程, 那么tgid则是统一的tgid
        	p->tgid = current->tgid;
        } else {
        	if (clone_flags & CLONE_PARENT)
        		p->exit_signal = current->group_leader->exit_signal;
        	else
        		p->exit_signal = (clone_flags & CSIGNAL);
        	p->group_leader = p;
                // 如果不是线程, tgid就是其自己的pid
        	p->tgid = p->pid;
        }


    // 省略代码

    
    
    }


wake_up_new_task
======================

注释上说就是唤醒新建的task

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
        // task的状态
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

         // 把task放到cpu的runqueue中
    	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
    #endif
    	rq = __task_rq_lock(p, &rf);
    	update_rq_clock(rq);
    	post_init_entity_util_avg(&p->se);
    
    	activate_task(rq, p, ENQUEUE_NOCLOCK);
    	p->on_rq = TASK_ON_RQ_QUEUED;
    	trace_sched_wakeup_new(p);
    	check_preempt_curr(rq, p, WF_FORK);
    #ifdef CONFIG_SMP
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


关于task调度, 参考linux_kernel/linux_task_schedule.rst

