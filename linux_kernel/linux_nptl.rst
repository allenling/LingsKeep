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

.. [12] http://www.cnblogs.com/hazir/p/linux_kernel_pid.htm

.. [13] https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces

.. [14] http://lzz5235.github.io/2016/01/11/copy_process.html

.. [15] http://blog.csdn.net/zhanglei4214/article/details/6765913

参考4是fork的一些解释

参考6有Linux下Thread的历史介绍

参考7中对pthread的创建过程源码有一部分跟4.15的有区别, 注意一下

参考8有关于ARCH_CLONE的解释, 以及clone_flags等其他参数的一些解释

参考9是pthread下的同步机制

参考11是task结构中, pids这个属性的一些解释. 注意的是4.15类型变成了pid_link而不是文章中的pid_type, 但是都是使用hash表结构.

参考12是内核中pid整个层级的设计, 4.15删除了pidmap一类的函数, 要注意一下

参考13是关于pid namespace

参考14, 15是copy_process这个函数的一些解释, 15更详细一点

**这里不涉及task调度, 调度参考linux_kernel/linux_task_schedule.rst**

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

可以看到第二列是pid, 第四列是lwp, 线程和进程分别对应各自的lwp, 然后进程的lwp和pid一致, 线程的pid和lwp是不一致的.

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

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L2110
    #ifdef __ARCH_WANT_SYS_FORK
    SYSCALL_DEFINE0(fork)
    {
    #ifdef CONFIG_MMU
        // 这里直接调用_do_fork, 传入的flags是SIGHLD
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

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L2130
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

getpid
-----------

因此, 调用getpid返回的pid其实是tgid(thread group id), 所以ps命令返回的lwp是task的pid, 而pid那一列则是tgid

  *Thread groups were a feature added in Linux 2.4 to support the POSIX threads notion of a set of threads that share a single PID.  Internally, this shared PID is the  so-called  thread  group
  identifier (TGID) for the thread group.  Since Linux 2.4, calls to getpid(2) return the TGID of the caller.*
  
  --- man clone

所以, 每一个进程和线程都指向一个task, 而每一个task都有自己的pid, 这个pid是内核看到的, 用来调度的, 而用户看到的pid则是tgid, 而ps命令根据参数决定是否返回

同一个tgid下的所有task(线程), 还是只返回tgid等于pid的task(主线程/进程)


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

pthread这个结构太长, 先放着吧

pthread_create/createthread
==================================

例如python中, 创建线程就直接调用pthread_create了, 而pthread_create会调用到createthread去实际创建线程

pthread_create代码在glibc/nptl/pthread_create.c

该函数一开始是在nptl/createthread.c中, 然后根据ChangeLog.18, 被移动到平台相关目录下

该函数会调用clone, 但是是根据平台不同调用不同的clone的. 

glibc/sysdeps/unix/sysv/linux/createthread.c

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

task结构属性很多, 下面通过clone的代码流程去了解创建线程的时候, task的属性赋值流程.

主要的属性有:

1. pid号(pid_t类型)和pids双链表(存储pid结构, 不是pid号), 内核中根据该链表去获取对应的task结构
   这里的pid号是task结构的, 也就是内核中每一个task都有自己的pid(叫pid是因为内核之前只有进程而没有线程), 但是
   现在称为tid可能更合适一些.

2. thread_info, thread_group, thread_info是该task的一些标志位, 比如是否有待处理信号, 则是通过该标志位是否置位有关, thread_group是线程的链表
   而thread_group是一个双链表结构, 如果是创建线程, 那么会把task的thread_group加入到主线程的thread_group中.

3. tgid, 也就是thread group id, 就是我们ps出来的pid, 同一个进程的线程们tgid都是主线程的pid, 用户看到的pid就是这个tgid

4. signal, sighand, shared_pending, blocked, pending, 和信号处理有关, signal.shared_pending线程组的待处理信号队列
   而pending是每个task自己的signal处理队列, 可以看成每一个线程自己的信号处理队列

pid结构和命名空间
=====================

都来自参考 [13]_

pid namespace是为了隔离进程的, 用来做虚拟化的等等, 比如docker等等工具, Google App Engine这些云平台.

*To create a new PID namespace, one must call the clone() system call with a special flag CLONE_NEWPID.*

1. CLONE_NEWPID

clone的时候传入CLONE_NEWPID将会新建一个pid namespace, 如果传入CLONE_NEWPID|CLONE_SIGCHLD, 那么子进程将自己分化出自己的namespace, 如果只传入

CLONE_SIGCHLD而不传入CLONE_NEWPID, 那么就是一个父子进程而子进程不会创建自己新的namespace

2. CLONE_NEWNET

这个是网络虚拟化, 也就是说, 传入这个标志, 则子进程和父进程都将"看到"所有的端口, 甚至都有自己的回环地址(loopback).

*In order to provide a usable network interface in the child namespace, it is necessary to set up additional “virtual” network interfaces which span multiple namespaces.*

*Finally, to make the whole thing work, a “routing process” must be running in the global network namespace to receive traffic from the physical interface, and route it through the appropriate virtual interfaces to to the correct child network namespaces.*

上面是说要构建虚拟网络, 还必须需要一个路由进程把物理的流量发送到指定的namespace下

*To do this by hand, you can create a pair of virtual Ethernet connections between a parent and a child namespace by running a single command from the parent namespace:
ip link add name veth0 type veth peer name veth1 netns <pid>*

在父子namespace之间, 创建一对虚拟以太网连接

所以, 一个task会有很多个pid(不同的namespace), 所以pid结构保存了这些信息


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/pid.h#L53
    struct upid {
        // namespace下的pid号
    	int nr;
        // 哪个namespace
    	struct pid_namespace *ns;
    };
    
    struct pid
    {
    	atomic_t count;
    	unsigned int level;
    	/* lists of tasks that use this pid */
        // tasks是一个hash表, 该hash表每一个类型都指向一个该类型的task结构的数组
    	struct hlist_head tasks[PIDTYPE_MAX];
    	struct rcu_head rcu;
    	struct upid numbers[1];
    };

upid是该pid结构, 在不同的namespace下, 对应的不同的数字, 而pid结构中, 保存了自己的upid的数组. 也就是全局的task, 其pid数字是全局唯一的, 但是在不同的namespace下, 可以相同

namespace中, 父层级不知道子层级, 子层级则保存了父层级

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/pid_namespace.h#L24
    struct pid_namespace {
        // 其他的属性先省略

        // 这个是存储pid号/结构的地方, 是一个radix tree(基数树)结构
    	struct idr idr;
        // 哪个层级
        unsigned int level;
        // 以及上一级namespace
        struct pid_namespace *parent;
        // 已分配了多少个pid
        unsigned int pid_allocated;

        // 其他的属性先省略
    } __randomize_layout;


从pid获取task
=================

通过pid号, 拿到pid结构, 再拿到task结构, 可以通过信号的处理来看看

在使用kill发送信号的时候, kill调用

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1399
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
        // 省略代码
    }

其中kill_pid_info的最后一个参数是pid结构, 然后通过传入的pid结构拿到task结构

.. code-block:: c


    // https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1313
    int kill_pid_info(int sig, struct siginfo *info, struct pid *pid)
    {
    	int error = -ESRCH;
    	struct task_struct *p;
    
    	for (;;) {
    	    rcu_read_lock();
    	    p = pid_task(pid, PIDTYPE_PID);
            // 省略代码
        }
        // 省略代码
     }


所以是

1. find_vpid, 拿到pid号对应的pid结构

2. pid_task, 通过pid结构, 以及传入的task类型, 获取对应的task结构 


find_vpid
---------------

这个操作基本上是去当前task的namespace下的idr(基数树)查找对应的pid号下的pid结构

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L244
    struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
    {
        // idr的查找
    	return idr_find(&ns->idr, nr);
    }
    EXPORT_SYMBOL_GPL(find_pid_ns);
    
    struct pid *find_vpid(int nr)
    {
    	return find_pid_ns(nr, task_active_pid_ns(current));
    }
    EXPORT_SYMBOL_GPL(find_vpid);

pid_nr拿到pid结构的pid号(全局)
================================

在copy_process中, 我们会看到, 先分配了一个新的pid结构, 然后再获取新pid结构的全局pid号

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/pid.h#L165
    static inline pid_t pid_nr(struct pid *pid)
    {
    	pid_t nr = 0;
    	if (pid)
            // 注意这里的numbers是拿第一个元素, 也就是下标是0的元素
            // 也就是全局的upid
    	    nr = pid->numbers[0].nr;
    	return nr;
    }



pid_task
------------

这个去是task结构中的tasks指向的hash表中, 根据传入的类型, 找到该第一个task(有点绕听起来)

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L305
    struct task_struct *pid_task(struct pid *pid, enum pid_type type)
    {
    	struct task_struct *result = NULL;
    	if (pid) {
    		struct hlist_node *first;
    		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
    					      lockdep_tasklist_lock_is_held());
    		if (first)
    			result = hlist_entry(first, struct task_struct, pids[(type)].node);
    	}
    	return result;
    }
    EXPORT_SYMBOL(pid_task);

其中hlist_first_rcu表示获取链表的第一个元素, 而链表的表头是pid->tasks[type], 也就是pid结构下tasks指向的hash表中对应type的元素

而hlist_entry就是通过计算task结构中node, 也就是task中包含的pids这个数组, 的偏移量去返回对应的task结构

**在copy_process中有具体的处理, 继续看下面**


分配一个pid
==============

新建一个pid结构的时候, 全局一个, 然后其每一个层级, 也就是父namespace, 都要映射一个

**注意的是, 这里只是分配新的pid而已, 并没有把pid和task对应起来, 对应起来是上一层, 也就是copy_process做的事情**

所以, 这里只是把pid结构中的tasks属性初始化而已

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L147
    struct pid *alloc_pid(struct pid_namespace *ns)
    {
    	struct pid *pid;
    	enum pid_type type;
    	int i, nr;
    	struct pid_namespace *tmp;
    	struct upid *upid;
    	int retval = -ENOMEM;
    
        // 分配一个pid结构
    	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
    	if (!pid)
    		return ERR_PTR(retval);
    
    	tmp = ns;
    	pid->level = ns->level;
    
        // 下面的for循环就是映射到每一个namespace层级上去
    	for (i = ns->level; i >= 0; i--) {
    		int pid_min = 1;
    
    		idr_preload(GFP_KERNEL);
    		spin_lock_irq(&pidmap_lock);
    
    		/*
    		 * init really needs pid 1, but after reaching the maximum
    		 * wrap back to RESERVED_PIDS
    		 */
    		if (idr_get_cursor(&tmp->idr) > RESERVED_PIDS)
    			pid_min = RESERVED_PIDS;
    
    		/*
    		 * Store a null pointer so find_pid_ns does not find
    		 * a partially initialized PID (see below).
    		 */
                // 当前循环的namespace的pid号则是
                // 从idr这个结构中分配出来的, 是可以复用的
    		nr = idr_alloc_cyclic(&tmp->idr, NULL, pid_min,
    				      pid_max, GFP_ATOMIC);
    		spin_unlock_irq(&pidmap_lock);
    		idr_preload_end();
    
    		if (nr < 0) {
    			retval = nr;
    			goto out_free;
    		}
    
                // pid的numbers这个数组的每一个元素都是upid 
                // 其中, nr被赋值为第i个层级的pid号码, 然后ns保存的时候对应的namespace
    		pid->numbers[i].nr = nr;
    		pid->numbers[i].ns = tmp;
                // 每次循环之后, 切换到父层级的namespace
    		tmp = tmp->parent;
    	}
    
    	if (unlikely(is_child_reaper(pid))) {
    		if (pid_ns_prepare_proc(ns))
    			goto out_free;
    	}
    
    	get_pid_ns(ns);
        // 该pid对应的计数为1
    	atomic_set(&pid->count, 1);
        // 初始化该pid的tasks这个数组中
        // 每一个类型的双向链表
    	for (type = 0; type < PIDTYPE_MAX; ++type)
    		INIT_HLIST_HEAD(&pid->tasks[type]);
    
    	upid = pid->numbers + ns->level;
    	spin_lock_irq(&pidmap_lock);
    	if (!(ns->pid_allocated & PIDNS_ADDING))
    		goto out_unlock;
        // 最后, 每一个namespace上, 真正把新建的pid结构加入到对应namespace的idr结构中
    	for ( ; upid >= pid->numbers; --upid) {
    		/* Make the PID visible to find_pid_ns. */
    		idr_replace(&upid->ns->idr, pid, upid->nr);
                // namespace中已分配的个数(pid_allocated)加1
    		upid->ns->pid_allocated++;
    	}
    	spin_unlock_irq(&pidmap_lock);
    
    	return pid;
    
    out_unlock:
    	spin_unlock_irq(&pidmap_lock);
    	put_pid_ns(ns);
    
    out_free:
    	spin_lock_irq(&pidmap_lock);
    	while (++i <= ns->level)
    		idr_remove(&ns->idr, (pid->numbers + i)->nr);
    
    	/* On failure to allocate the first pid, reset the state */
    	if (ns->pid_allocated == PIDNS_ADDING)
    		idr_set_cursor(&ns->idr, 0);
    
    	spin_unlock_irq(&pidmap_lock);
    
    	kmem_cache_free(ns->pid_cachep, pid);
    	return ERR_PTR(retval);
    }

1. 分配pid的原则是每一个namespace都要指定, 例如当前namespace, 父namespace, 然后父亲的父亲等等层级

2. 每一个namespace分配的pid号码, 则是通过idr_alloc_cyclic这个函数去实现

3. 分配之后, 保存在pid这个结构的numbers数组中

4. 注意的是, 在for循环里面只是新建了对应namespace的pid数字, 然后在最后的for循环里面才会把
   对应的namespace下, 对应的pid数字对应的pid结构加入到其idr属性上


idr_alloc_cyclic
=================

通过注释可知, 先找一个大于last id的id, 不存在, 则找最小的, 有效的id

所以称为循环(cyclic)找嘛, 也就是id值会复用

显然, 在alloc_pid中, 传入的pid_min是1, end就是pid_max, pid_max是可配置的了

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/lib/idr.c#L49
    /**
     * idr_alloc_cyclic - allocate new idr entry in a cyclical fashion
     * @idr: idr handle
     * @ptr: pointer to be associated with the new id
     * @start: the minimum id (inclusive)
     * @end: the maximum id (exclusive)
     * @gfp: memory allocation flags
     *
     * Allocates an ID larger than the last ID allocated if one is available.
     * If not, it will attempt to allocate the smallest ID that is larger or
     * equal to @start.
     */
    int idr_alloc_cyclic(struct idr *idr, void *ptr, int start, int end, gfp_t gfp)
    {
    	int id, curr = idr->idr_next;
    
        // start和curr谁大, 谁大从谁开始分配
    	if (curr < start)
    		curr = start;
        // 找到一个比当前大的id号, 当然是可用的
    	id = idr_alloc(idr, ptr, curr, end, gfp);
    	if ((id == -ENOSPC) && (curr > start))
                // 找不到, 从start开始找
    		id = idr_alloc(idr, ptr, start, curr, gfp);
    
        // 下一个则是当前id + 1
    	if (id >= 0)
    		idr->idr_next = id + 1U;
    
    	return id;
    }
    EXPORT_SYMBOL(idr_alloc_cyclic);

加入start=1, 也就是alloc_pid中的传参, 那么找不到比idr当前大的, 可用的pid数字, 那么就从start开始, 也就是从1开始找, 也就是

和注释上的流程.

获取task的pid
================

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L334
    struct pid *get_task_pid(struct task_struct *task, enum pid_type type)
    {
    	struct pid *pid;
    	rcu_read_lock();
    	if (type != PIDTYPE_PID)
    		task = task->group_leader;
    	pid = get_pid(rcu_dereference(task->pids[type].pid));
    	rcu_read_unlock();
    	return pid;
    }
    EXPORT_SYMBOL_GPL(get_task_pid);

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/pid.h#L76
    static inline struct pid *get_pid(struct pid *pid)
    {
    	if (pid)
    		atomic_inc(&pid->count);
    	return pid;
    }

get_task_pid则强制拿到PIDTYPE_PID类型的task, 返回PIDTYPE_PID类型的task中, pids这个数组指定的type的元素

**有点绕呀有点绕~~~~~~~**



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
    // 注释上就是说, 创建一个新的task就是复制一份老的
    // 然后启动的操作交给调用者
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

        if (!p)
        	goto fork_out;
        
        /*
         * This _must_ happen before we call free_task(), i.e. before we jump
         * to any of the bad_fork_* labels. This is to avoid freeing
         * p->set_child_tid which is (ab)used as a kthread's data pointer for
         * kernel threads (PF_KTHREAD).
         */
        // 下面是CLONE_CHILD_SETTID和CLONE_CHILD_CLEARTID标志位
        p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
        /*
         * Clear TID on mm_release()?
         */
        p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr : NULL;

        
        // 省略代码
        // 初始化task的pending队列
        // 初始化的意思就是把队列置空
        init_sigpending(&p->pending);

        // 省略代码

        /* Perform scheduler related setup. Assign this task to a CPU. */
        // 这里复制调度相关的属性, 包括调度类, 调度优先级等等
        // 线程/子进程都是从主线程/父进程继承过来的, 这里也就是复制一份属性
        retval = sched_fork(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_policy;

        // 省略代码

        // 复制文件
        retval = copy_files(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_semundo;

        // 复制文件描述符(fd)
        retval = copy_fs(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_files;

        // 复制信号操作函数
        retval = copy_sighand(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_fs;
        
        // 这里会根据是否是线程去决定是否公用信号结构
        retval = copy_signal(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_sighand;

        // 省略代码

        // 复制IO!!!
        retval = copy_io(clone_flags, p);
        if (retval)
            goto bad_fork_cleanup_namespaces;

        retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);
        if (retval)
        	goto bad_fork_cleanup_io;
        
        if (pid != &init_struct_pid) {
                // !!!!!!!!这里去新建了pid结构
                // !!!!!!!!但是下面的pid_nr才会去把pid和task给对应起来!!!
        	pid = alloc_pid(p->nsproxy->pid_ns_for_children);
        	if (IS_ERR(pid)) {
        		retval = PTR_ERR(pid);
        		goto bad_fork_cleanup_thread;
        	}
        }

        // 省略代码

        
        // 这个是拿到pid结构中全局的pid号码
        p->pid = pid_nr(pid);
        // 下面是针对线程, 赋值task结构里面的属性
        // 包括什么tgid呀
        if (clone_flags & CLONE_THREAD) {
                // !!!!!注意一下这个exit_signal = -1
                // 后面会使用到, 说明新建的task不是thread group leader
        	p->exit_signal = -1;
                // 注意这里, group_leader则是当前线程的group_leader
        	p->group_leader = current->group_leader;
                // 如果是线程, 那么tgid则是统一的tgid
        	p->tgid = current->tgid;
        } else {
        	if (clone_flags & CLONE_PARENT)
        		p->exit_signal = current->group_leader->exit_signal;
        	else
        		p->exit_signal = (clone_flags & CSIGNAL);
                // 如果不是创建线程, 那么group_leader则是自己
        	p->group_leader = p;
                // 如果不是线程, tgid就是其自己的pid
        	p->tgid = p->pid;
        }

        // 省略代码

        // 初始化线程组链表, 其实就是next=prev=head
        INIT_LIST_HEAD(&p->thread_group);

        // 省略代码

        // 这里一般都会走if里面的代码
        if (likely(p->pid)) {
        	ptrace_init_task(p, (clone_flags & CLONE_PTRACE) || trace);
        
                // 把pid结构放入到task中, pids这个数组对应的type的位置中
                // 这个需要和attch_pid一起看
        	init_task_pid(p, PIDTYPE_PID, pid);

                // thread_group_leader的判断是: p->exit_signal >= 0;
                // 之前如果带入的flags有CLONE_THREAD的话, 那么p->exit_signal会被复制为-1的
                // 所以不会走if里面的代码
        	if (thread_group_leader(p)) {
                    // 线程不会走这里
        	} else {
        	    current->signal->nr_threads++;
        	    atomic_inc(&current->signal->live);
        	    atomic_inc(&current->signal->sigcnt);

                    // !!!!!!!!!!把task加入到group_leader的thread_group链表
        	    list_add_tail_rcu(&p->thread_group,
        	    		  &p->group_leader->thread_group);
        	    list_add_tail_rcu(&p->thread_node,
        	    		  &p->signal->thread_head);
        	}
                // 这里就比较绕了
                // 这里是把p加入到p>tasks[type]这个链表中
                // 这个需要和init_task_pid一起看
        	attach_pid(p, PIDTYPE_PID);
        	nr_threads++;
        }

        // 后面还有一堆代码, 先这样吧
    
    
    }

init_task_pid/attach_pid
==========================

这两个比较绕一点, 简单来说是前一个把pid放入到task中, 而第二个是把task放入到pid中, 互相包含方便快速查找

查找就是一个container_of的计算了


init_task_pid的操作

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L1506
    static inline void
    init_task_pid(struct task_struct *task, enum pid_type type, struct pid *pid)
    {
    	 task->pids[type].pid = pid;
    }

也就是

.. code-block:: python

    '''
                 链表头
    task +-----> pids   +-----+ PIDTYPE_PGID
                              |
                              + PIDTYPE_PID  +---+ node
                                                 |      
                                                 |      
                                                 + pid 
                                                   |
    new_pid_struct <-------------------------------+
    
    '''

而attach_pid

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L259
    void attach_pid(struct task_struct *task, enum pid_type type)
    {
    	struct pid_link *link = &task->pids[type];
        // 注意的是, 最后一个参数才是head, 第一个参数是要加入的node
        hlist_add_head_rcu(&link->node, &link->pid->tasks[type]);
    }

也就是

.. code-block:: python

    '''
                 链表头
    task +-----> pids   +-----+ PIDTYPE_PGID
                              |
                              + PIDTYPE_PID  +---+ node >-->----+
                                                 |              |
                                                 |              |
                                                 + pid          |
                                                   |            |
    new_pid_struct <-------------------------------+            |
         |                                                      |
         |                                                      |
         +-----+ upid                                           |
               |                                                |
               |                                                |
               + tasks +--+ PIDTYPE_PID ---> node1 --> node2 -> + (注意的是, node一般是PIDTYPE_PID下的第一个元素, 这里写多个是表示该结构是一个链表)
                          |
                          + PIDTYPE_PGID

    
    '''

一般, 进程的task结构回事pid结构中的tasks中的第一个元素, 所以pid_task函数的做法就是:

1. 根据namespace(一般是current的namespace)和pid数字, 拿到idr中, pid数字对应的pid结构

2. 1中拿到的就是上一个图的new_pid_struct, 然后拿到tasks对应type的第一个元素, 就是进程的task结构了


再来看看find_vpid和pid_task的代码

.. code-block:: c

    // 这里拿到task的pid, 然后拿到namespace
    struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)
    {
    	return ns_of_pid(task_pid(tsk));
    }
    EXPORT_SYMBOL_GPL(task_active_pid_ns);

    // 这里调用find_pid_ns, 传入task_active_pid_ns返回的namespace
    // 继续看下面
    struct pid *find_vpid(int nr)
    {
    	return find_pid_ns(nr, task_active_pid_ns(current));
    }
    EXPORT_SYMBOL_GPL(find_vpid);

    // 这里通过namespace和nr, 也就是pid号, 拿到namespace中idr结构对应的pid结构
    struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
    {
    	return idr_find(&ns->idr, nr);
    }
    EXPORT_SYMBOL_GPL(find_pid_ns);

    // 这里通过pid拿到的是task结构
    struct task_struct *pid_task(struct pid *pid, enum pid_type type)
    {
    	struct task_struct *result = NULL;
    	if (pid) {
    		struct hlist_node *first;
    		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
    					      lockdep_tasklist_lock_is_held());
    		if (first)
    			result = hlist_entry(first, struct task_struct, pids[(type)].node);
    	}
    	return result;
    }
    EXPORT_SYMBOL(pid_task);

所以, pid结构中已经记住了task, 所以直接拿就好了


dup_task_struct
====================

dup_task_struct函数会去调用平台相关的arch_dup_task_struct函数, x86下是在


但其实也没做什么特别的, 只是把task结构复制一份, 然后改一下stack等等.

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/fork.c#L512 
    static struct task_struct *dup_task_struct(struct task_struct *orig, int node)
    {
        // 省略代码

        // 分配栈
        stack = alloc_thread_stack_node(tsk, node);
        if (!stack)
        	goto free_tsk;
        
        stack_vm_area = task_stack_vm_area(tsk);
        
        // 平台相关的复制task结构
        err = arch_dup_task_struct(tsk, orig);
        // 省略代码
        // 后面大都是跟栈相关的操作
    }

    // https://elixir.bootlin.com/linux/v4.15/source/arch/x86/kernel/process.c#L94
    // x86下的复制task结构
    int arch_dup_task_struct(struct task_struct *dst, struct task_struct *src)
    {
    	memcpy(dst, src, arch_task_struct_size);
    #ifdef CONFIG_VM86
    	dst->thread.vm86 = NULL;
    #endif
    
    	return fpu__copy(&dst->thread.fpu, &src->thread.fpu);
    }

wake_up_new_task
======================

注释上说就是唤醒新建的task

**这里需要参考linux_task_schedule.rst**

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



