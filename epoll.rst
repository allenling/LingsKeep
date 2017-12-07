select,epoll
=============

一般了解
---------

When descriptors are added to an epoll instance, they can be added in two modes: level triggered and edge triggered. When you use level triggered mode, and data is available for reading, epoll_wait(2) will always return with ready events. If you don't read the data completely, and call epoll_wait(2) on the epoll instance watching the descriptor again, it will return again with a ready event because data is available. In edge triggered mode, you will only get a readiness notfication once. If you don't read the data fully, and call epoll_wait(2) on the epoll instance watching the descriptor again, it will block because the readiness event was already delivered.

epoll的两种模式, ET和LT的区别. LT是若你没有完全的读完数据, wait仍然会返回, RT是不管你读没读玩数据,wait只返回一次.

所以,若我们在ET模式下,wait返回后只读了一半的数据,然后再次调用wait,则这时候会阻塞,而在LT模式下,再次调用wait,则会马上返回,因为还有数据没读完.

ET模式下只要有数据到达就触发,但是只是触发一次.

一道腾讯后台开发的面试题
使用Linuxepoll模型，水平触发模式；当socket可写时，会不停的触发socket可写的事件，如何处理？

第一种最普遍的方式：
需要向socket写数据的时候才把socket加入epoll，等待可写事件。
接受到可写事件后，调用write或者send发送数据。
当所有数据都写完后，把socket移出epoll。

这种方式的缺点是，即使发送很少的数据，也要把socket加入epoll，写完后在移出epoll，有一定操作代价。

一种改进的方式：
开始不把socket加入epoll，需要向socket写数据的时候，直接调用write或者send发送数据。如果返回EAGAIN(缓冲区满)，把socket加入epoll，在epoll的驱动下写数据，全部数据发送完毕后，再移出epoll。

这种方式的优点是：数据不多的时候可以避免epoll的事件处理，提高效率。

https://segmentfault.com/a/1190000004597522

LT的处理过程:

. accept一个连接，添加到epoll中监听EPOLLIN事件

. 当EPOLLIN事件到达时，read fd中的数据并处理

. 当需要写出数据时，把数据write到fd中；如果数据较大，无法一次性写出，那么在epoll中监听EPOLLOUT事件

. 当EPOLLOUT事件到达时，继续把数据write到fd中；如果数据写出完毕，那么在epoll中关闭EPOLLOUT事件

ET的处理过程:

. accept一个一个连接，添加到epoll中监听EPOLLIN|EPOLLOUT事件

. 当EPOLLIN事件到达时，read fd中的数据并处理，read需要一直读，直到返回EAGAIN为止

. 当需要写出数据时，把数据write到fd中，直到数据全部写完，或者write返回EAGAIN

. 当EPOLLOUT事件到达时，继续把数据write到fd中，直到数据全部写完，或者write返回EAGAIN

同步,异步,阻塞,非阻塞的区别
--------------------------------

http://www.cnblogs.com/Anker/p/5965654.html

select, poll, epoll的区别
------------------------------

http://www.cnblogs.com/Anker/p/3265058.html

http://blog.csdn.net/kai8wei/article/details/51233494

1. select的时候, 每次都要传递我们要监听的fd, 这个时候就是每次都要把fd列表从用户空间拷贝到内核控件, 而epoll一开始就把所以的fd都拷贝到内核了, 不用每次都拷贝一次, 然后当有新的fd需要监听的时候

   epoll_ctl调用直接把新的fd加入到内核空间中. 并且, epoll在内核中的保存区是一个高速缓存(cache), 是一个红黑树来支持高速添加, 查找, 删除操作.

2. 每次内核都是遍历一遍所有的fd, 然后返回哪些fd已经就绪. 而epoll在创建的时候, 就为每个fd添加了一个回调函数, 这个回调函数会在fd就绪的时候, 将就绪的fd加入到就绪列表(内核中是链表)中

   epoll_wait就是遍历这个列表而已.

3. python版本的select和select系统调用有点区别
   
   3.1 select的系统调用中, 返回值是一个就绪fd的个数, 所以还是需要你自己去遍历三个列表, 哪个fd就绪了.

   3.2 select会修改fd集合, 比如read_fds中监听了三个fd, fd1, fd2, fd3, 将read_fds传给select, 然后fd1受信, 则read_fds中就只有fd2,fd3都被置为0;

       摘自wiki中select条目的example:

        .. code-block:: c

            if (-1 == (nready = select(maxfd+1, &readfds, NULL, NULL, NULL)))
                die("select()");
            //这里返回的就是个数, 可以看打印的内容
            (void)printf("Number of ready descriptor: %d\n", nready);
            //然后必须遍历fd集合
            for (i=0; i<=maxfd && nready>0; i++)
            {
                //readfds中未受信的fd被设为0, 所以我们必须逐个判断哪些fd被受信了
                if (FD_ISSET(i, &readfds))
                {
                    nready--;
      
       而python版本的select.select会返回三个列表, 三个列表代表可读的fd列表, 可写的fd列表已经其他情况的fd的列表.
       不需要你去遍历原fd列表去看看哪个fd就绪了, 并且不会修改传入的fd列表.

4. python版本的epoll和epoll_wait系统调用有点区别.

epoll_wait会返回就绪fd的个数, 跟select一样, 但是epoll_wait还会返回包含就绪fd构造体的的数组, 每一个元素都是epoll_event的结构.

epoll_event的构造体定义有data和event来拿过部分:

.. code-block:: c

    typedef union epoll_data {
        void        \*ptr;
        int          fd;
        uint32_t     u32;
        uint64_t     u64;
    } epoll_data_t;

    struct epoll_event {
        uint32_t     events;      // Epoll events, 这里就是EPOLLIN等event类型
        epoll_data_t data;        // User data variable
    };

epoll_wait系统调用定义为:

.. code-block:: c

    int epoll_wait(int epfd, struct epoll_event \*events, int maxevents, int timeout)

第二个参数就是就绪数组.

下面摘抄man手册的例子:

.. code-block:: c

   struct epoll_event ev, events[MAX_EVENTS];
   //调用epoll_wait
   nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
   //遍历就绪数组 
   for (n = 0; n < nfds; ++n) {
       //直接拿就绪数组中的epoll_event结构, 判断events[n].events & EPOLLIN等可以知道event类型
       if (events[n].data.fd == listen_sock) {

python版本的epoll返回值就是[(fd1, event1), (fd2, event2), ...]的形式.
        
所以epoll在大多数情况是空闲的时候, 比起select快很多, 若fd大多数都是就绪的时候, 跟select比起来, 差不多, 因为此时内核需要遍历的就绪列表跟全部fd就差不多了.

如此，一颗红黑树，一张准备就绪fd链表，少量的内核cache，就帮我们解决了大并发下的fd（socket）处理问题。

1 执行epoll_create时，创建了红黑树和就绪list链表。

2 执行epoll_ctl时，如果增加fd（socket），则检查在红黑树中是否存在，存在立即返回，不存在则添加到红黑树上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪list链表中插入数据。

3 执行epoll_wait时立刻返回准备就绪链表里的数据即可。


epoll流程源码
====================

总结起来就是: 大概明白流程，但是对其中的细节不明白，比如vfs的poll实现, 各种wait_queue, epi, pwq, eventpoll等结构的一些字段的用处等等

还有epollevent里面为什么有了ready list, 还需要ovflist?


看到哪里写到哪里吧~~

https://www.slideshare.net/llj098/epoll-from-the-kernel-side

https://idndx.com/2015/07/08/the-implementation-of-epoll-4/

linux v4.15-rc2 https://github.com/torvalds/linux/blob/master/fs/eventpoll.c, line:1992

其中只是摘抄部分代码, 省略了其他代码


epoll_ctl
---------------

SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd, struct epoll_event __user \*, event)

https://idndx.com/2015/07/08/the-implementation-of-epoll-1/

.. code-block:: c

    {

	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;
    
        f = fdget(epfd);
        
        tf = fdget(fd);

        // 这里的private_data就是一个struct eventpoll结构
        ep = f.file->private_data;

    }

1. epfd就是epoll实例(struct eventpoll结构体)对应的fd, 通过epfd可以找到对应的epoll实例(下面的流程), 而epoll实例也保存了自己的fd

2. 通过epfd获取eqfd对应的文件结构f, 通过传入的fd获取, 称为target fd, tfd, 获取tfd对应的文件结构tf

   其中文件结构是struct file, 称为文件实例, fget在: https://elixir.free-electrons.com/linux/v4.15-rc2/source/include/linux/file.h#L43

   fget是通过文件描述符拿到对应的文件结构

3. 通过f.file->private_data这句是之前通过epfd拿到对应的文件实例，文件实例中的private_data就是文件实例对应的具体数据结构, 在这里就是epoll实例.
   
   由于linux中所有的对象都是文件，比如tty设备什么的都是文件, 但是每一个对象都有自己具体的数据结构，这个就是private_data了: http://tsecer.blog.163.com/blog/static/15018172012225103242956/

4. 还从用户空间把用户传入的epoll_event结构体拷贝到内核区了: copy_from_user(&epds, event, sizeof(struct epoll_event))), 这里epds就是拷贝了用户传入的epoll_event结构体event.

struct eventpoll的定义:

http://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L186

.. code-block:: c

    struct eventpoll {

    	spinlock_t lock;
    
    	struct mutex mtx;
    
    	wait_queue_head_t wq;
    
    	wait_queue_head_t poll_wait;
    
    	struct list_head rdllist;
    
        // 红黑树
    	struct rb_root_cached rbr;
    
    	struct epitem *ovflist;
    
    	struct wakeup_source *ws;
    
    	struct user_struct *user;
    
    	struct file *file;
    
    	int visited;
    	struct list_head visited_list_link;
    
    #ifdef CONFIG_NET_RX_BUSY_POLL
    	unsigned int napi_id;
    #endif
    };



ep_find
-------------

初始化之后, 查找epoll中是否保存了目标fd

epoll实例中保存了一颗红黑树(ep.rbr), 用于快速查找

.. code-block:: c

    {
        ep = f.file->private_data;

        epi = ep_find(ep, tf.file, fd);
    }

其中ep_find定义为:  

.. code-block:: c

    static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
    {
    	int kcmp;
    	struct rb_node *rbp;
    	struct epitem *epi, *epir = NULL;
    	struct epoll_filefd ffd;
    
    	ep_set_ffd(&ffd, file, fd);
    	for (rbp = ep->rbr.rb_root.rb_node; rbp; ) {
    		epi = rb_entry(rbp, struct epitem, rbn);
    		kcmp = ep_cmp_ffd(&ffd, &epi->ffd);
    		if (kcmp > 0)
    			rbp = rbp->rb_right;
    		else if (kcmp < 0)
    			rbp = rbp->rb_left;
    		else {
    			epir = epi;
    			break;
    		}
    	}
    
    	return epir;
    }

红黑树保存的是epitem这个结构体: https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L142

查找的时候, 构建epitem对应的struct epoll_filefd, 然后比对, struct epoll_filefd定义为:

.. code-block:: c

    struct epoll_filefd {
    	struct file *file;
    	int fd;
    } __packed;

epoll_filefd保存的是文件实例和对应的fd数字, 说白了对比的时候就是用这两个来对比, 对比函数为ep_cmp_ffd:


.. code-block:: c

    static inline int ep_cmp_ffd(struct epoll_filefd *p1,
    			     struct epoll_filefd *p2)
    {
    	return (p1->file > p2->file ? +1:
    	        (p1->file < p2->file ? -1 : p1->fd - p2->fd));
    }

这个函数的意思是( https://idndx.com/2014/09/01/the-implementation-of-epoll-1/ ):

ep_cmp_ffd() first compares the memory address of the struct file, the one with higher address will be considered as "bigger".

首先比对的是文件实例的内存地址, 地址高的就比较大

If the memory address are the same, which is possible since multiple file descriptors could be referring to the same struct file (for example, via dup()), ep_cmp_ffd() will simply consider the file with higher file descriptor number as "bigger". By doing this, it is guaranteed that for any two nonequivalent file descriptor, ep_cmp_ffd() can compare them. And for two file descriptors that are the same, ep_cmp_ffd() will return 0.

如果内存地址一样, 也就是两个文件实例是一样的, 也有可能fd不一样. 这也是可能的, 因为不同的fd可以对应同一个文件实例, 比如dup操作. 然后fd数值大的就比较大.



ep_insert
--------------

https://idndx.com/2014/09/02/the-implementation-of-epoll-2/

.. code-block:: c

    epi = ep_find(ep, tf.file, fd);

    switch (op) {
    	case EPOLL_CTL_ADD:
    		if (!epi) {
    			epds.events |= POLLERR | POLLHUP;
    			error = ep_insert(ep, &epds, tf.file, fd, full_check);
    		} else
    			error = -EEXIST;
    		if (full_check)
    			clear_tfile_check_list();
    		break;
    }


如果在红黑树中没有找到fd, 那么调用ep_insert插入

.. code-block:: c

    static int ep_insert(struct eventpoll \*ep, struct epoll_event \*event, struct file \*tfile, int fd, int full_check)
    {
        
    }


其中先初始化epoll_event对应的struct epitem, 称为epi

.. code-block:: c

    /* Item initialization follow here ... */
    INIT_LIST_HEAD(&epi->rdllink);
    INIT_LIST_HEAD(&epi->fllink);
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);
    epi->event = *event;
    epi->nwait = 0;
    epi->next = EP_UNACTIVE_PTR;


然后初始化struct ep_pqueue, 这个ep_pqueue只是wrap一下poll_table和epi:

.. code-block:: c

    //https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L254
    struct ep_pqueue {
    	poll_table pt;
    	struct epitem *epi;
    };

下面是初始化poll_table的_qproc
.. code-block:: c

    // epq是struct ep_pqueue
    epq.epi = epi;
    // 这一句的作用只是把ep_ptable_queue_proc赋值到poll_table._qproc
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
    
    revents = ep_item_poll(epi, &epq.pt, 1);
    



关于poll_table, 原文讲得我也不是很清楚, 貌似就是一个设计标准, 就是linux kernel vfs中poll(这个poll不是系统调用select, poll那个poll)实现的一部分

然后接着是的ep_item_poll, ep_item_poll里面对调用一个poll_wait的函数, 这个函数会调用poll_table里面的_qproc

.. code-block:: c

    // https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L877
    static unsigned int ep_item_poll(struct epitem *epi, poll_table *pt, int depth)
    {
        poll_wait(epi->ffd.file, &ep->poll_wait, pt);
    }

poll_wait:

.. code-block:: c

    static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
    {
    	if (p && p->_qproc && wait_address)
                // 调用poll_table->_qproc
    		p->_qproc(filp, wait_address, p);
    }


之前epi对应的poll_table中绑定的_qproc是ep_ptable_queue_proc, 所以调用的是ep_ptable_queue_proc


.. code-block:: c

    static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
    				 poll_table *pt)
    {
    	struct epitem *epi = ep_item_from_epqueue(pt);
        // 初始化一个eppoll_entry对象
    	struct eppoll_entry *pwq;
    
    	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
                // 把ep_poll_callback绑定到pwq->wait->func
    		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
                // pwq记住wait_queue_head_t, 这个传入的whead是epoll实例中的poll_wait
    		pwq->whead = whead;
                // pwq记住epoll实例
    		pwq->base = epi;
                // 下面的add_wait_queue和add_wait_queue_exclusive都是把pwq->wait加入到epoll->poll_wait链表中
                // 可以这么简单的理解: 这个pwq加入到了观察列表
                if (epi->event.events & EPOLLEXCLUSIVE)
    			add_wait_queue_exclusive(whead, &pwq->wait);
    		else
    			add_wait_queue(whead, &pwq->wait);
    		list_add_tail(&pwq->llink, &epi->pwqlist);
    		epi->nwait++;
    	} else {
    		/* We have to signal that an error occurred */
    		epi->nwait = -1;
    	}
    }

ep_ptable_queue_proc就是生成一个pwq, 然后加入pwq->wait加入到epoll->poll_wait, 表示开始监听这个pwq, 其中pwq->wait是wait_queue_entry_t, 而

epoll->poll_wait是wait_queue_head_t, 从名字上就知道前者是后面的一个成员(entry). 然后每当这个wait_queue_entry_t被唤醒的时候，调用的就是绑定的

ep_poll_callback, ep_poll_callback会唤醒sleep的epoll_wait的~~

pwq->wait might be the most important thing in the whole epoll implementation because it will be used for:

1. monitoring events happening on that particular monitored file

2. wake up other processes as necessary


然后会把epi加入红黑树

.. code-block:: c

    revents = ep_item_poll(epi, &epq.pt, 1);

    ep_rbtree_insert(ep, epi);


所以, 简单来说, ep_insert的流程是初始化epi, pwq, 然后把pwq绑定到epoll实例的poll_wait这个等待链表中, 然后如果等待链表有成员受信, 则会调用ep_poll_callback, 这个ep_poll_callback会把

受信的epi添加到, epoll实例中的等待列表rdllist和ovflist上, 然后调用epoll_wait的时候检查到rdlist和ovflist不为空, 就是有fd受信, 则拷贝rdlist到用户空间.

这里如果epoll_wait是处于sleep状态, 那么ep_poll_callback被调用的时候会唤醒epoll_wait的, 也就是调用epoll_wait的进程了.


**关于kerbel vfs poll实现:**

绑定的ep_poll_callback是在某个fd有事件的时候所调用的回调, 对某个fd进行监视是否有数据是kernel vfs的poll实现, 简单来说就是

在kernel中所有的对象都是文件, 文件有open, read/write, close, poll等接口实现, 然后private_data就是文件对象的具体结构, 比如epoll对应的fd的private_data就是struct eventepoll,

而poll是在文件对象上的一个监视, 系统调用select, poll都会使用到上面的poll实现.


epoll_wait
~~~~~~~~~~~~~~~

https://idndx.com/2014/09/22/the-implementation-of-epoll-3/

.. code-block:: c

    // https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L2148
    SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
    		int, maxevents, int, timeout)
    {
        error = ep_poll(ep, events, maxevents, timeout);
    }

epoll_wait就是先各种初始化, 然后调用ep_poll函数

.. code-block:: c

    // https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L1736
    static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
    {

    	if (timeout > 0) {
                //　如果有timeout, 则这里会设置最后的时间
		struct timespec64 end_time = ep_set_mstimeout(timeout);

		slack = select_estimate_accuracy(&end_time);
                // to就是最后的时间了
		to = &expires;
		*to = timespec64_to_ktime(end_time);
	} else if (timeout == 0) {
		/*
		 * Avoid the unnecessary trip to the wait queue loop, if the
		 * caller specified a non blocking operation.
		 */
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
                // 如果没有timeout, 则会直接去检查是否有event受信
		goto check_events;
	}
        //　接下来有fetch_events和check_events两个代码块
        fetch_events:
        check_events:

    }

**fetch_events:**

.. code-block:: c


    fetch_events:
    
        if (!ep_events_available(ep)) {
        	ep_reset_busy_poll_napi_id(ep);
        
        	/*
        	 * We don't have any available event to return to the caller.
        	 * We need to sleep here, and we will be wake up by
        	 * ep_poll_callback() when events will become available.
        	 */
                // 上面的注释就是说我们要把当前进程设置为TASK_INTERRUPTIBLE, 这样ep_poll_callback可以唤醒等待的进程

                // 这里是把当前进程(current是个全局宏, 表示当前进程)绑定到wait_queue_entry->private中, 这样当前进程就是可以被唤醒了
        	init_waitqueue_entry(&wait, current);
                // 把当前的wait加入到ep->wq中
        	__add_wait_queue_exclusive(&ep->wq, &wait);
        
        	for (;;) {
        		/*
        		 * We don't want to sleep if the ep_poll_callback() sends us
        		 * a wakeup in between. That's why we set the task state
        		 * to TASK_INTERRUPTIBLE before doing the checks.
        		 */
                         // 
                        // 设置进程状态
        		set_current_state(TASK_INTERRUPTIBLE);
        		/*
        		 * Always short-circuit for fatal signals to allow
        		 * threads to make a timely exit without the chance of
        		 * finding more events available and fetching
        		 * repeatedly.
        		 */
        		if (fatal_signal_pending(current)) {
        			res = -EINTR;
        			break;
        		}
                        // 这里回去检查是否有受信的epi, 检查的方式就是!list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
                        // 或者timed_out
        		if (ep_events_available(ep) || timed_out)
        			break;
        		if (signal_pending(current)) {
        			res = -EINTR;
        			break;
        		}
        
        		spin_unlock_irqrestore(&ep->lock, flags);
                        // schedule_hrtimeout_range这里就是sleep的过程了, 返回值是如果过期, 也就是达到了to依然没有被唤醒, 则是1
                        // 如果是被唤醒的话，就EINTR
                        // #define	EINTR		 4	/* Interrupted system call */
        		if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
        			timed_out = 1;
        
        		spin_lock_irqsave(&ep->lock, flags);
        	}
        
        	__remove_wait_queue(&ep->wq, &wait);
                // 设置当前状态为TASK_RUNNING, 那么可以被调度运行
        	__set_current_state(TASK_RUNNING);
        }


所以fetch_events就是sleep直到超时或者有受信发生


**check_events:**

.. code-block:: c

    check_events:
    	/* Is it worth to try to dig for events ? */
    	eavail = ep_events_available(ep);
    
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	/*
    	 * Try to transfer events to user space. In case we get 0 events and
    	 * there's still timeout left over, we go trying again in search of
    	 * more luck.
    	 */
    	if (!res && eavail &&
    	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
    		goto fetch_events;
    
    	return res;

check_events就是检查是否有受信, 然后把受信发送到用户空间: ep_send_events

ep_send_events回去遍历ready list的

.. code-block:: c

    // https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L1697
    static int ep_send_events(struct eventpoll *ep,
    			  struct epoll_event __user *events, int maxevents)
    {
    	struct ep_send_events_data esed;
    
    	esed.maxevents = maxevents;
    	esed.events = events;
    
        // 遍历ready list
    	return ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
    }

ep_poll_callback
-------------------

https://idndx.com/2014/09/22/the-implementation-of-epoll-3/


epoll_wait的唤醒是依赖于ep_poll_callback的(至少代码注释上是这么说的)

.. code-block:: c

    // https://elixir.free-electrons.com/linux/v4.15-rc2/source/fs/eventpoll.c#L1114

    static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
    {

        // 下面对把epi添加到ep->ovflist中
        if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
        	if (epi->next == EP_UNACTIVE_PTR) {
        		epi->next = ep->ovflist;
        		ep->ovflist = epi;
        		if (epi->ws) {
        			/*
        			 * Activate ep->ws since epi->ws may get
        			 * deactivated at any time.
        			 */
        			__pm_stay_awake(ep->ws);
        		}
        
        	}
        	goto out_unlock;
        }
        
        /* If this file is already in the ready list we exit soon */
        // 把epi->rdllink加入到ep->rdllist中
        if (!ep_is_linked(&epi->rdllink)) {
        	list_add_tail(&epi->rdllink, &ep->rdllist);
        	ep_pm_stay_awake_rcu(epi);
        }

    }





