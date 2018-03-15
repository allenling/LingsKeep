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

.. [11] http://www.cnblogs.com/parrynee/archive/2010/01/14/1648152.html

参考4是fork的一些解释

参考6有Linux下Thread的历史介绍

参考7中对pthread的创建过程源码有一部分跟4.15的有区别, 注意一下

参考8有关于ARCH_CLONE的解释, 以及clone_flags等其他参数的一些解释

参考9是pthread下的同步机制

参考11是task结构中, pids这个属性的一些解释. 注意的是4.15类型变成了pid_link而不是文章中的pid_type, 但是都是使用hash表结构.

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
    
    // 省略代码, 一堆属性设置
    
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

task结构
============

task结构属性很多, 下面借助clone的代码去了解创建线程的时候, task结构属性的赋值流程.


clone中新建task结构
=====================

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

