epoll
=========

内核版本v4.15

.. [1] https://idndx.com/2014/09/01/the-implementation-of-epoll-1/, 这个系列有4章

.. [2] http://blog.csdn.net/kai8wei/article/details/51233494

.. [3] https://stackoverflow.com/questions/19942702/the-difference-between-wait-queue-head-and-wait-queue-in-linux-kernel

.. [4]  http://www.xml.com/ldd/chapter/book/ch05.html, Going to Sleep and Awakening和A Deeper Look at Wait Queues这两节

.. [5] http://guojing.me/linux-kernel-architecture/posts/wait-queue/

.. [6] https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/

.. [7] https://www.cnblogs.com/Anker/p/7071849.html

.. [8] https://blog.csdn.net/mumumuwudi/article/details/50552470

.. [9] https://github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898

.. [10] https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/

参考6和7是惊群的参考, 而参考8中, 说内核4.5已经解决惊群了, 但是参考8中提到的那个内核的patch(也就是参考9)是说加入

add_wait_queue_exclusive, 但是add_wait_queue_exclusive只是加入WQ_FLAG_EXCLUSIVE标志位, 并且在

__wake_up_common中只是判断WQ_FLAG_EXCLUSIVE标志位而已, 也就是只是保证同一时间只有一个task被唤醒而已

我感觉参考8中想说的标志位是EPOLLEXCLUSIVE标志位, 这个标志位在参考6中提到可以使用EPOLLEXCLUSIVE标志位去解决epoll监听accept的时候发生惊群问题

**所以, 参考8中的代码截图有误导~~~截取的代码补全或者说不对**

参考10是参考6的第二部分, 讲述了epoll在close/dup的问题



小结
======

epoll的man手册里面的例子(man epoll):

.. code-block:: c

    // events这个数组是存储内核拷贝过来的就绪的event
    struct epoll_event ev, events[MAX_EVENTS];

    // 一个无限循环去获取受信的fd
    for (;;) {

        //调用epoll_wait
        nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);

        //遍历就绪数组
        for (n = 0; n < nfds; ++n) {
            // events数组中每一个元素包含一个data
            // data.fd就是受信的fd了, 直接使用就好
            // 而data.events就是哪个事件, 读, 写, 或者读写(read | write)
            // select的话需要遍历整个列表, 然后判断是否受信
            if (events[n].data.fd == listen_sock) {
                
            }
        }
    }

对比下select的man手册里面的例子(man select):

.. code-block:: c

      int main(void)
       {
           fd_set rfds;
           struct timeval tv;
           int retval;

           /* Watch stdin (fd 0) to see when it has input. */

           // 监听读的fd集合
           FD_ZERO(&rfds);
           FD_SET(0, &rfds);

           /* Wait up to five seconds. */

           tv.tv_sec = 5;
           tv.tv_usec = 0;

           // 只返回个数
           retval = select(1, &rfds, NULL, NULL, &tv);
           /* Don't rely on the value of tv now! */

           if (retval == -1)
               perror("select()");
           else if (retval)
               printf("Data is available now.\n");
               // 参照注释, 你需要遍历列表, 然后
               // 调用FD_ISSET来判断是否fd是否受信
               /* FD_ISSET(0, &rfds) will be true. */
           else
               printf("No data within five seconds.\n");

           exit(EXIT_SUCCESS);
       }


1. select每次调用都要拷贝数据到内核, epoll不是

2. select每次都需要自己去遍历列表, 哪些fd受信了, epoll会返回给你已受信的fd, 并且还有受信的event是什么.


epoll在大多数情况是空闲的时候, 比起select快很多, 若fd大多数都是就绪的时候, 跟select比起来, 差不多, 因为此时内核需要遍历的就绪列表跟全部fd就差不多了.

一颗红黑树，一张准备就绪fd链表，少量的内核cache，就帮我们解决了大并发下的fd处理问题。

1. 执行epoll_create时，创建了红黑树和就绪链表(ready list).

2. 执行epoll_ctl时，如果增加fd则检查在红黑树中是否存在，存在立即返回，不存在则添加到红黑树上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪list链表中插入数据.

3. 执行epoll_wait时立刻返回准备就绪链表里的数据即可.

4. 复制就绪链表到用户态的时候, 复制是操作的是就绪链表的一份copy, 然后把就绪链表置空, 因为有锁, 所以感觉就算多个task调用ep_poll的时候问题也不大

   也就是txlist = copy(rdllist), 然后rdllist = [], 然后操作txlist

   操作txlist的时候, 会把遍历过的, 满足条件的元素删除, 然后最后可能txlist会变为空, txlist = [], 或者因为LT/ET模式, txlist不为空

   然后把ovlist中的event加入到rdllist, 然后再把txlist加入rdllist, 此时rdllist又有可能不是空了

   如果不为空, 则调用wake_up_locked再次去唤醒监听epoll的task

参考: http://blog.csdn.net/kai8wei/article/details/51233494

rcu
=====

linux中的rcu(Read-Copy Update)机制: https://www.ibm.com/developerworks/cn/linux/l-rcu/

关于WRITE_ONCE的解释: https://stackoverflow.com/questions/34988277/write-once-in-linux-kernel-lists, 没怎么看懂

linux vfs
============

linux中的vfs是指一套统一的接口, 然后任何实现了该接口的fs都能被挂载到linux, 然后用户态/内核态都可以使用统一的接口去操作file.

vfs还处理了page cache, inode cache, buffer cache等等. vfs是内核的和物理存储交互的一个软件层(layer), 只定义接口, 具体的操作交给具体fs的实现(ext2,3,4, tmpfs等等)

可以把vfs类比于wsgi去理解.



linux wait_queue
====================

  A *wait queue* is exactly that -- a queue of processes that are waiting for an event.
  
  --- 参考2

更多wait_queue查看参考 [3]_, 参考 [4]_, 参考 [5]_

关于休眠, 有sleep_on/sleep_on_timeout和interruptible_sleep_on/interruptible_sleep_on_timeout两组系统调用, 不同的地方是, 前者是不可中断的, 后面是可中断的.

也就是前者必须得等到设置到的时间/或者等待的event受信的时候会"醒过来", 而后者则是可以在没有到设定时间的时候, 发送一个中断, 让其"醒过来".

**wait_queue中的唤醒不一定是真正的唤醒操作, 而是调用wait_queue中的元素, 每一个元素都是wait_queue_entry结构, 中的定义的回调. 至于是不是真正的去"唤醒"线程, 由回调决定**


event_poll
==============

这是epoll结构体

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L186


.. code-block:: c

    struct eventpoll {
    	/* Protect the access to this structure */
        // epoll的自旋锁
    	spinlock_t lock;
    
    	/*
    	 * This mutex is used to ensure that files are not removed
    	 * while epoll is using them. This is held during the event
    	 * collection loop, the file cleanup path, the epoll file exit
    	 * code and the ctl operations.
    	 */
        // 操作红黑树中的fd(也包括查询)的时候必须要获取这个锁
    	struct mutex mtx;
    
    	/* Wait queue used by sys_epoll_wait() */
        // 这个是调用epoll_wait的时候, 把当前进程加入到wq这个wait_queue中
    	wait_queue_head_t wq;
    
    	/* Wait queue used by file->poll() */
        // 而这个是epoll自己的wait_queue
        // 可以类比于socket自己的wait_queue
    	wait_queue_head_t poll_wait;
    
    	/* List of ready file descriptors */
        // list_head是一个双链表结构
        // 只包含两个属性, prev和next
        // 所以这个ready链表是一个双链表
    	struct list_head rdllist;
    
    	/* RB tree root used to store monitored fd structs */
        // 红黑树的根节点
    	struct rb_root_cached rbr;
    
    	/*
    	 * This is a single linked list that chains all the "struct epitem" that
    	 * happened while transferring ready events to userspace w/out
    	 * holding ->lock.
    	 */
    	struct epitem *ovflist;
    
    	/* wakeup_source used when ep_scan_ready_list is running */
    	struct wakeup_source *ws;
    
    	/* The user that created the eventpoll descriptor */
        // 当前用户的信息结构
        // 其中包含了一个epoll_watches属性, user->epoll_watches, 表示当前用户
        // 监听了几个fd, 可以设置一个最大监听数
        // 每次add一个fd到epoll中, 那么这个个数就加1
    	struct user_struct *user;
    
        // epoll对应的file结构
        // 很多时候是通过file的fd获得file, 继而获取epoll
        // fd->file->epoll
    	struct file *file;
    
    	/* used to optimize loop detection check */
    	int visited;
    	struct list_head visited_list_link;
    
    #ifdef CONFIG_NET_RX_BUSY_POLL
    	/* used to track busy poll napi_id */
    	unsigned int napi_id;
    #endif
    };

有两个wait_queue_head_t, wq和poll_wait

1. wq是调用epoll_poll的是, 把当前进程放入wq中, 一旦有event受信, 则唤醒wq中的进程.

2. poll_wait, 根据注释, 就是epoll自己的poll实现使用的wait_queue, 因为epoll也实现了poll操作, 所以是支持poll行为的. 可类比于socket的wait_queue, 具体下面有解释

有两个就绪链表, rdllist和ovflist

1. rdlist是把epoll把受信的event发送给用户态的时候, 遍历的已受信的链表

2. 而ovflist则是为了无锁复制

   如果现在epoll正在发送event到用户态, 此时则正在受信的时间暂时放在ovflist中, 当epoll处理完rdllist的时候, 会把ovflist的event加入到rdllist中.

   也就是ovflist是为了不影响正在处理的rdllist, 暂时存放受信event的地方. 主要是发送event到用户态的时候是无锁状态(不会拿epoll中的lock这个自旋锁), 所以为了避免"污染"rdllist, 又没有拿锁, 则只能

   用一个临时链表来解决. 无锁是为了效率.

ovflist参考: http://blog.csdn.net/mercy_pm/article/details/51381216, https://idndx.com/2015/07/08/the-implementation-of-epoll-4/


epitem
========

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L142

.. code-block:: c

    struct epitem {

        // 这里保存了对应的红黑树节点
    	union {
    		/* RB tree node links this structure to the eventpoll RB tree */
    		struct rb_node rbn;
    		/* Used to free the struct epitem */
    		struct rcu_head rcu;
    	};
    
    	/* List header used to link this structure to the eventpoll ready list */
    	struct list_head rdllink;
    
    	/*
    	 * Works together "struct eventpoll"->ovflist in keeping the
    	 * single linked chain of items.
    	 */
    	struct epitem *next;
    
    	/* The file descriptor information this item refers to */
    	struct epoll_filefd ffd;
    
    	/* Number of active wait queue attached to poll operations */
    	int nwait;
    
    	/* List containing poll wait queues */
    	struct list_head pwqlist;
    
    	/* The "container" of this item */
    	struct eventpoll *ep;
    
    	/*. List header used to link this item to the "struct file" items list */
    	struct list_head fllink;
    
    	/* wakeup_source used when EPOLLWAKEUP is set */
    	struct wakeup_source __rcu *ws;
    
    	/* The structure that describe the interested events and the source fd */
    	struct epoll_event event;
    };

1. 保存红黑树节点的作用是: 查询红黑树的时候, 可以通过已知的红黑树节点的地址通过计算内存其在epitem中的地址偏移量, 反过来得到epitem的地址(参考ep_find)

2. ffd是epitem对应的fd的结构, ffd保存了fd和file两个结构, 红黑树查找的时候, 就是比对ffd, 也即是比对file和fd来确定对应的fd是否存在于红黑树

3. rdlllink是一旦epitem受信了, 那么会把rdllink加入到epoll中的rdllist的尾部

4. pwq结构是和eppoll_entry有关, 看后面

epoll_create1
===============

新建一个epoll结构体, 然后用一个fd指向epoll结构体, 然后返回这个fd.

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L1936

.. code-block:: c

    /*
     * Open an eventpoll file descriptor.
     */
    SYSCALL_DEFINE1(epoll_create1, int, flags)
    {
    	int error, fd;
        // epoll结构体
    	struct eventpoll *ep = NULL;
    	struct file *file;
    
    	/* Check the EPOLL_* constant for consistency.  */
    	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);
    
    	if (flags & ~EPOLL_CLOEXEC)
    		return -EINVAL;
    	/*
    	 * Create the internal data structure ("struct eventpoll").
    	 */
    	error = ep_alloc(&ep);
    	if (error < 0)
    		return error;
    	/*
    	 * Creates all the items needed to setup an eventpoll file. That is,
    	 * a file structure and a free file descriptor.
    	 */
        // 新建一个fd
    	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
    	if (fd < 0) {
    		error = fd;
    		goto out_free_ep;
    	}
        // 绑定ep到file->private_data
    	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
    				 O_RDWR | (flags & O_CLOEXEC));
    	if (IS_ERR(file)) {
    		error = PTR_ERR(file);
    		goto out_free_fd;
    	}
        // 然后ep的file指向file
        // 这样ep和其file就互相指向了, 通过其中一个都能获取另一个
    	ep->file = file;

        // 绑定fd和file的关系
        // 让fd指向file
    	fd_install(fd, file);
    	return fd;
    
    out_free_fd:
    	put_unused_fd(fd);
    out_free_ep:
    	ep_free(ep);
    	return error;
    }

event_poll_fops
======================

event_poll_fops是一套ep定义的file操作接口, 其实就是原生的文件操作接口

file_operations包含的就是vfs的标准接口的集合

http://elixir.free-electrons.com/linux/v4.15/source/include/linux/fs.h#L1692

.. code-block:: c

    // 定义了read, write等文件操作接口
    struct file_operations {
    	struct module *owner;
    	loff_t (*llseek) (struct file *, loff_t, int);
    	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);

        // 这里是poll接口
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        // 这里省略了其他接口
    }

    // 说明event_poll_fops也是一般性的文件操作接口
    // 也是一个file_operations结构体
    // http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L315
    static const struct file_operations eventpoll_fops;


    // 然后eventpoll_fops后面又修改了一下属性
    // http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L964
    static const struct file_operations eventpoll_fops = {
    #ifdef CONFIG_PROC_FS
    	.show_fdinfo	= ep_show_fdinfo,
    #endif
        // 直接覆盖了下面三个函数
    	.release	= ep_eventpoll_release,
    	.poll		= ep_eventpoll_poll,
    	.llseek		= noop_llseek,
    };

所以epoll本身也是一个支持poll的文件, 其poll函数是ep_eventpoll_poll.

anon_inode_getfile
======================

anon_inode_getfile就是生成一个file结构, 然后把file->private_data指向event_poll(第三个传参)

下面是anon_inode_getfile的关于private_data的代码

.. code-block:: c

    struct file *anon_inode_getfile(const char *name,
    				const struct file_operations *fops,
    				void *priv, int flags)
    {
    
        // 这里有一些分配inode等工作, 先省略掉
    
        // 分配一个file结构
        // 包含了传入的接口结构体
    	file = alloc_file(&path, OPEN_FMODE(flags), fops);
    	if (IS_ERR(file))
    		goto err_dput;
    	file->f_mapping = anon_inode_inode->i_mapping;
    
    	file->f_flags = flags & (O_ACCMODE | O_NONBLOCK);
            // 这里绑定private_data到传入的priv参数
            // epoll_create1中就是event_poll对象
    	file->private_data = priv;
    
    	return file;
    }


所以关系就是
===============

.. code-block:: python

    ''' 

                                        fd
                                         
                              fd指向file |
                                         |    +-->其他
                                              |
                +--->file -----------> file --+-->private_data
                |                                     
                |                                     |
    event_poll--+-->其他                              |
                                                      |
       |                                              |
       +<--file的private_data指向ep-------------------+      
    
    '''

由于event_poll和file都各自有指向对方, 所以从其中一个都能获取另外一个



epoll_ctl
============

epoll_ctl是对fd进行插入, 删除已经修改的接口.

如果是插入操作, 那么插入的fd对应的file必须支持poll操作.

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L1992

.. code-block:: c

    // 传参的顺序是: epoll对应的fd, 操作码, 操作的fd, 操作的fd对应的epoll_event对象
    SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
    		struct epoll_event __user *, event)
    {
    	int error;
    	int full_check = 0;
    	struct fd f, tf;
    	struct eventpoll *ep;
    	struct epitem *epi;
    	struct epoll_event epds;
    	struct eventpoll *tep = NULL;
    
    	error = -EFAULT;
        // 注意, 这里会把用户态的epoll_event复制到epds中, 是一个epoll_event结构
        // ep_op_has_event操作是判断op是否是删除操作, 不是的话复制
    	if (ep_op_has_event(op) &&
    	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
    		goto error_return;
    
    	error = -EBADF;
        // 获取epoll对应的fd对应的file
    	f = fdget(epfd);
    	if (!f.file)
    		goto error_return;
    
    	/* Get the "struct file *" for the target file */
        // 用户指定的fd对应的file
    	tf = fdget(fd);
    	if (!tf.file)
    		goto error_fput;
    
    	/* The target file descriptor must support poll */
        // 如果要操作的fd对应的file不支持poll操作, 报错
    	error = -EPERM;
    	if (!tf.file->f_op->poll)
    		goto error_tgt_fput;
    
    	/* Check if EPOLLWAKEUP is allowed */
        // 这里op如果不是删除操作, 那么epoll_event加入wake的flag
    	if (ep_op_has_event(op))
    		ep_take_care_of_epollwakeup(&epds);
    
        /*
        * We have to check that the file structure underneath the file descriptor
        * the user passed to us _is_ an eventpoll file. And also we do not permit
        * adding an epoll file descriptor inside itself.
        */
        // 这里是判断
        // 1. 操作的fd不能是epoll本身
        // 2. is_file_epoll是检查是否file的接口和event_poll_fops一样
        // 所以就是检查fd对应是否也是epoll本身
        error = -EINVAL;
        if (f.file == tf.file || !is_file_epoll(f.file))
        	goto error_tgt_fput;
            
        /*
         * epoll adds to the wakeup queue at EPOLL_CTL_ADD time only,
         * so EPOLLEXCLUSIVE is not allowed for a EPOLL_CTL_MOD operation.
         * Also, we do not currently supported nested exclusive wakeups.
         */
        // 第一个判断是op是否不是删除操作, 第二个判断是用户是否加入了EPOLLEXCLUSIVE标志
        // 这里的操作就是modify不能使用EPOLLEXCLUSIVE标志
        // 并且add操作不支持嵌套的exclusive唤醒
        if (ep_op_has_event(op) && (epds.events & EPOLLEXCLUSIVE)) {
        	if (op == EPOLL_CTL_MOD)
        		goto error_tgt_fput;
        	if (op == EPOLL_CTL_ADD && (is_file_epoll(tf.file) ||
        			(epds.events & ~EPOLLEXCLUSIVE_OK_BITS)))
        		goto error_tgt_fput;
        }

    
    	/*
    	 * At this point it is safe to assume that the "private_data" contains
    	 * our own data structure.
    	 */
        // 通过file获取到了epoll对象
    	ep = f.file->private_data;
    
        // 下面是针对add操作的一个判断操作, 没看懂, 先省略吧
    
    	/*
    	 * Try to lookup the file inside our RB tree, Since we grabbed "mtx"
    	 * above, we can be sure to be able to use the item looked up by
    	 * ep_find() till we release the mutex.
    	 */
        // 从红黑树中查找fd
    	epi = ep_find(ep, tf.file, fd);
    
    	error = -EINVAL;
    	switch (op) {
    	case EPOLL_CTL_ADD:
                // 如果epitem不存在红黑树中, 调用insert
    		if (!epi) {
    			epds.events |= POLLERR | POLLHUP;
    			error = ep_insert(ep, &epds, tf.file, fd, full_check);
    		} else
                // 否则报已存在错误
    			error = -EEXIST;
    		if (full_check)
    			clear_tfile_check_list();
    		break;
               // 下面是删除和修改的操作, 先省略
    	}
        // 省略下面的错误处理
    }

1. 检查操作码.

2. 如果不是删除操作, 那么把用户态的 **epoll_event** 拷贝到内核态.

3. 检查操作的fd是否有效, 有效则调用ep_find去查找epoll中红黑树是否包含该fd.

4. 调用插入等操作函数.

fd有效条件包括:

1. 不能是epoll本身, 也就是不能把epoll加入到自己中, 强调自己是因为epoll对应的fd也可以加入到其他epoll中, 因为

   epoll对应的fd也继承了event_poll_fops这些操作.

2. fd对应的file一定实现有poll操作.

ep_op_has_event
=================

这个是判断op的操作是否是删除, 不是删除操作就需要把user传入的epoll_event结构复制到内核态

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L362


.. code-block:: c

    // 看注释吧
    /* Tells if the epoll_ctl(2) operation needs an event copy from userspace */
    static inline int ep_op_has_event(int op)
    {
    	return op != EPOLL_CTL_DEL;
    }


ep_find
==========

从epoll中的红黑树中找到是否有传入的fd

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L1041

.. code-block:: c

    /*
     * Search the file inside the eventpoll tree. The RB tree operations
     * are protected by the "mtx" mutex, and ep_find() must be called with
     * "mtx" held.
     */
    // 注释说, 操作红黑树的时候必须获取epoll中的mtx这个锁, 这是一个互斥锁
    static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
    {
    	int kcmp;
    	struct rb_node *rbp;
    	struct epitem *epi, *epir = NULL;
    	struct epoll_filefd ffd;
    
    	ep_set_ffd(&ffd, file, fd);

        // 这个循环就是查找红黑树
        // rb_right和rb_left就是红黑树的子节点
    	for (rbp = ep->rbr.rb_root.rb_node; rbp; ) {
                // 这一句是从红黑树的节点中
                // 获取对应的epitem结构
                // 因为epitem结构中第一个属性就是rbn
                // 这里直接可以返回epitem的地址
    		epi = rb_entry(rbp, struct epitem, rbn);

                // 比较epitem的是比较epitem->ffd
    		kcmp = ep_cmp_ffd(&ffd, &epi->ffd);
    		if (kcmp > 0)
    			rbp = rbp->rb_right;
    		else if (kcmp < 0)
    			rbp = rbp->rb_left;
    		else {
                        // 找到了!!!
    			epir = epi;
    			break;
    		}
    	}
    
    	return epir;
    }


红黑树
===========

epoll中存放fd的结构是ep_item, 红黑树使得fd的查找最坏也能打到O(logN)

比较的时候需用组织成ffd结构, 然后通过ffd生成一个epitem结构(这里其实就是把ffd设置到epitem中, 当然还包括其他信息), 然后再比较epitem中的ffd.

其实ffd里面就包含两个属性, 一个file, 一个fd

rbp获取epitem
==================

由于epitem中保存了对应的rbp, 所以可以通过rbp获取对应的epitem:

.. code-block:: c

    epi = rb_entry(rbp, struct epitem, rbn);

    // rb_entry的定义: http://elixir.free-electrons.com/linux/v4.15/source/include/linux/rbtree.h#L66

    #define  rb_entry(ptr, type, member) container_of(ptr, type, member)
   
rb_entry最终调用到的时候container_of, 这个宏的意思是通过计算内存地址的偏移量, 可以通过属性得到整个结构体的地址.

比如epitem是包含了rbn属性的, 所以知道了rbn的地址, 可以计算rbn在整个结构体的偏移量, 得到epitem的地址.

container_of的参考: https://stackoverflow.com/questions/15832301/understanding-container-of-macro-in-the-linux-kernel


比较过程
============

epitem比较的时候是比较其中的ffd保存的file和fd

.. code-block:: c

   // 设置ffd, ffd->file, ffd.fd
   ep_set_ffd(&ffd, file, fd);

   // 红黑树的节点转成epitem结构
   epi = rb_entry(rbp, struct epitem, rbn);

   // 比较ffd和epitem的ffd
   kcmp = ep_cmp_ffd(&ffd, &epi->ffd);


ffd比较的时候先比较file, 再比较fd:

.. code-block:: c

    /* Compare RB tree keys */
    static inline int ep_cmp_ffd(struct epoll_filefd *p1,
    			     struct epoll_filefd *p2)
    {
    	return (p1->file > p2->file ? +1:
    	        (p1->file < p2->file ? -1 : p1->fd - p2->fd));
    }

也就是如果p1->file > p2->file, 那么返回+1, 反正进入到p1->file < p2->file的比较, 如果为真, 那么返回-1, 否则返回fd相减, fd相减也相当于比较了.

一开始是先比较文件地址, 文件地址比较高比较大, 如果文件地址一样, 但是有可能fd不一样, 比如使用dup操作, 是得两个fd指向同一个文件, 所以先比较

文件, 然后比较fd大小. 参考自: https://idndx.com/2014/09/01/the-implementation-of-epoll-1/



ep_insert
==========

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L1412

如果搜索不到fd, 那么执行插入操作 



.. code-block:: c

    static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
    		     struct file *tfile, int fd, int full_check)
    {
    	int error, revents, pwake = 0;
    	unsigned long flags;
    	long user_watches;
    	struct epitem *epi;
    	struct ep_pqueue epq;
    
        // 这里检查用户当前的watch数量
        // 如果设置了最大watch数量, 超过限制数量则报错
        // 后面会把该user_watches加1的
    	user_watches = atomic_long_read(&ep->user->epoll_watches);
    	if (unlikely(user_watches >= max_user_watches))
    		return -ENOSPC;

        // 分配一个epitem的结构
    	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
    		return -ENOMEM;
    
        // 下面各种双链表的初始化
        // 过程就是双链表的头next和prev都指向自己了
    	/* Item initialization follow here ... */
    	INIT_LIST_HEAD(&epi->rdllink);
    	INIT_LIST_HEAD(&epi->fllink);
    	INIT_LIST_HEAD(&epi->pwqlist);
        // epitem保存下对应的epoll结构
    	epi->ep = ep;
        // 设置epitem的ffd属性, 作为红黑树遍历的时候的比对属性
    	ep_set_ffd(&epi->ffd, tfile, fd);
        // 这个epitem是要监听的是什么事件
        epi->event = *event;

    	epi->nwait = 0;
    	epi->next = EP_UNACTIVE_PTR;
    	if (epi->event.events & EPOLLWAKEUP) {
    		error = ep_create_wakeup_source(epi);
    		if (error)
    			goto error_create_wakeup_source;
    	} else {
    		RCU_INIT_POINTER(epi->ws, NULL);
    	}
    
        // 下面是初始化ep_pqueue这个结构
    	/* Initialize the poll table using the queue callback */
    	epq.epi = epi;
    	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
    
    	/*
    	 * Attach the item to the poll hooks and get current event bits.
    	 * We can safely use the file* here because its usage count has
    	 * been increased by the caller of this function. Note that after
    	 * this operation completes, the poll callback can start hitting
    	 * the new item.
    	 */
        // ep_item的作用下面说
    	revents = ep_item_poll(epi, &epq.pt, 1);
    
    	/*
    	 * We have to check if something went wrong during the poll wait queue
    	 * install process. Namely an allocation for a wait queue failed due
    	 * high memory pressure.
    	 */
    	error = -ENOMEM;
    	if (epi->nwait < 0)
    		goto error_unregister;
    
    	/* Add the current item to the list of active epoll hook for this file */
    	spin_lock(&tfile->f_lock);
    	list_add_tail_rcu(&epi->fllink, &tfile->f_ep_links);
    	spin_unlock(&tfile->f_lock);
    
    	/*
    	 * Add the current item to the RB tree. All RB tree operations are
    	 * protected by "mtx", and ep_insert() is called with "mtx" held.
    	 */
        // 插入红黑树
    	ep_rbtree_insert(ep, epi);
    
    	/* now check if we've created too many backpaths */
    	error = -EINVAL;
    	if (full_check && reverse_path_check())
    		goto error_remove_epi;
    
    	/* We have to drop the new item inside our item list to keep track of it */
    	spin_lock_irqsave(&ep->lock, flags);
    
    	/* record NAPI ID of new item if present */
    	ep_set_busy_poll_napi_id(epi);
    
        // 之前的ep_item_poll是直接调用了epi对应的file的poll函数
        // 返回的revents大于0, 说明该event受信了, 直接把fd添加到epoll的就绪链表中
    	/* If the file is already "ready" we drop it inside the ready list */
    	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {

                // 把epi加入到epoll结构的rdllink的最后
    		list_add_tail(&epi->rdllink, &ep->rdllist);
    		ep_pm_stay_awake(epi);
    
    		/* Notify waiting tasks that events are available */
    		if (waitqueue_active(&ep->wq))
                        // 唤醒wq中休眠的进程
    			wake_up_locked(&ep->wq);
    		if (waitqueue_active(&ep->poll_wait))
    			pwake++;
    	}
    
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	atomic_long_inc(&ep->user->epoll_watches);
    
    	/* We have to call this outside the lock */
    	if (pwake)
    		ep_poll_safewake(&ep->poll_wait);
    
    	return 0;
    
        // 下面是错误处理, 忽略掉
    
    	return error;
    }

所以, ep_insert就是设置各种回调, 然后插入红黑树的过程, 最后去判断下就绪链表是否有值, 有值的话就去唤醒wq中的进程


init_poll_funcptr
====================

这个函数是设置poll_table结构中的回调函数, 然后把其_key属性设置为所有事件.

https://elixir.bootlin.com/linux/v4.15/source/include/linux/poll.h#L70


.. code-block:: c

    static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
    {
    	pt->_qproc = qproc;
    	pt->_key   = ~0UL; /* all events enabled */
    }

由于~0=-1, 然后-1的补码是11111...111, 所以是接收所有的event.

-1的原码是10000...001, 其反码是原码符号位不变, 其他1变0, 0变1, 所以是1111...1110, 然后补码是反码加1, 所以是11111...1111

所以, epoll_insert中

.. code-block:: c

   epq.epi = epi;
   init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);


就是把epq中的poll_table的回调设置为ep_ptable_queue_proc


.. code-block:: python

    '''
    
    epq(ep_pqueue) --+---> poll_table -+--->_qproc=ep_ptable_queue_proc
                     |                 |
                     |                 +--->_key=1111...1111
                     |
                     +--->epi(赋值为对应的epitem)
    
    '''


ep_item_poll
================

这里其实是主要作用是, 调用传入的epi对应的file的poll实现.

比如, 如果epi对应的是一个socket, 那么这里基本上就是调用socket的file的poll实现了

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L877

.. code-block:: c

    static unsigned int ep_item_poll(struct epitem *epi, poll_table *pt, int depth)
    {
    	struct eventpoll *ep;
    	bool locked;
    
        // poll_table中的_key, 也就是-1
    	pt->_key = epi->event.events;

        // 如果epi对应的file不是epoll, 则直接调用poll实现
        // 一般都是走这个if的return代码了
    	if (!is_file_epoll(epi->ffd.file))
    		return epi->ffd.file->f_op->poll(epi->ffd.file, pt) &
    		       epi->event.events;
    
        // 获得epoll结构
    	ep = epi->ffd.file->private_data;
        // 调用poll_wait
    	poll_wait(epi->ffd.file, &ep->poll_wait, pt);
    	locked = pt && (pt->_qproc == ep_ptable_queue_proc);
    
        // 调用ep_scan_ready_list
    	return ep_scan_ready_list(epi->ffd.file->private_data,
    				  ep_read_events_proc, &depth, depth,
    				  locked) & epi->event.events;
    }

**注意的是: 如果对应epi的file不是eventpoll结构, 则直接调用其file的poll实现然后返回**, 比如epi对应的file是socket的话, 那么就直接调用poll实现了.

**is_file_epoll** 这个函数是判断: f->f_op == &eventpoll_fops的, 所以, 比如socket, 那么必然不相等, 所以, 比如是调用if中的return语句, 也就是调用file对应的poll操作.

socket的poll参考 `这里 <https://github.com/allenling/LingsKeep/tree/master/linux_kernel/socket.rst>`_

所以, 大部分情况下, 都不会走到poll_wait中的.

**那么, 什么时候会调用后面的poll_wait呢?** 暂时不知道, 看代码就是只有epi的file是一个epoll的时候才会走后面, 也就是epoll监听的fd对应的也是一个epoll才行


poll_wait
===========

不管是谁的poll调用, 最后是会走到poll_wait这个函数的, 比如tcp_poll这个tcp socket的poll实现.

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/tcp.c#L496
    unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
    {
    	unsigned int mask;
    	struct sock *sk = sock->sk;
    	const struct tcp_sock *tp = tcp_sk(sk);
    	int state;
    
    	sock_rps_record_flow(sk);
    
    	sock_poll_wait(file, sk_sleep(sk), wait);
            // 省略代码
    }

    // 而sk_sleep是获取sock结构(不是socket结构)的wait_queue结构
    // https://elixir.bootlin.com/linux/v4.15/source/include/net/sock.h#L1692
    static inline wait_queue_head_t *sk_sleep(struct sock *sk)
    {
    	BUILD_BUG_ON(offsetof(struct socket_wq, wait) != 0);
        // 这里的wait是sock的wait_queue
    	return &rcu_dereference_raw(sk->sk_wq)->wait;
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/net/sock.h#L2000
    static inline void sock_poll_wait(struct file *filp,
    		wait_queue_head_t *wait_address, poll_table *p)
    {
    	if (!poll_does_not_wait(p) && wait_address) {
                // ---------------又回到了poll_wait这个函数
    		poll_wait(filp, wait_address, p);
    		/* We need to be sure we are in sync with the
    		 * socket flags modification.
    		 *
    		 * This memory barrier is paired in the wq_has_sleeper.
    		 */
    		smp_mb();
    	}
    }


而poll_wait则是一个linux中poll实现的通用接口, 实际上就是调用传入的poll_table中的设置回调函数

https://elixir.bootlin.com/linux/v4.15/source/include/linux/poll.h#L43

.. code-block:: c

    static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
    {
    	if (p && p->_qproc && wait_address)
    		p->_qproc(filp, wait_address, p);
    }

对于epoll, p->_qproc就是ep_ptable_queue_proc这个函数


ep_ptable_queue_proc
======================

这里初始化wait_queue_entry, 包括wait_queue_entry中的回调(func属性).

把wait_queue_entry加入到 **对应的file自己的wait_queue中**, 所以一旦file受信, 那么对每一个wait_queue_entry, 调用其func回调函数.

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L1231

.. code-block:: c

    static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
    				 poll_table *pt)
    {
        // 获取epitem
    	struct epitem *epi = ep_item_from_epqueue(pt);
    	struct eppoll_entry *pwq;
    
    	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
                // ------------初始化eppoll_entry中wait这个wait_queue_entry的回调
    		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
    		pwq->whead = whead;
    		pwq->base = epi;
                // ----------下面是把pwq中的wait_queue_entry加入到epoll结构的wait_queue列表中
    		if (epi->event.events & EPOLLEXCLUSIVE)
    			add_wait_queue_exclusive(whead, &pwq->wait);
    		else
    			add_wait_queue(whead, &pwq->wait);
                // 把pwq的llink加入到epi的pwqlist这个链表中
    		list_add_tail(&pwq->llink, &epi->pwqlist);
    		epi->nwait++;
    	} else {
    		/* We have to signal that an error occurred */
    		epi->nwait = -1;
    	}
    }

具体例子来说, 如果调用的是tcp socket的poll, 那么传入的whead就是sock结构(不是socket结构)的socket_wq属性中的wait属性, 其中wait是一个wait_queue

.. code-block:: python

    '''
                                    whead
                                    
                                    |whead是下面的wait
                                    |
    
           sock -+-->socket_wq +--->wait(wait_queue)
    
    '''

init_waitqueue_func_entry是把ep_poll_callback设置为pwq的wait的回调函数, pwq->wait是一个wait_queue_entry结构 

https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L87

.. code-block:: c


    static inline void
    init_waitqueue_func_entry(struct wait_queue_entry *wq_entry, wait_queue_func_t func)
    {
    	wq_entry->flags		= 0;
    	wq_entry->private	= NULL;
        // 这里就是ep_poll_callback
    	wq_entry->func		= func;
    }

关于EPOLLEXCLUSIVE, 这个配置是解决epoll惊群问题的:

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/uapi/linux/eventpoll.h#L44
    #define EPOLLEXCLUSIVE (1U << 28)

这样不管是read还是write, and操作EPOLLEXCLUSIVE都是真, add_wait_queue_exclusive调用是把wait_queue_entry设置上WQ_FLAG_EXCLUSIVE标志, 这样唤醒的时候, 只会唤醒一个.

.. code-block:: c

    void add_wait_queue_exclusive(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
    {
    	unsigned long flags;
    
        // 设置上WQ_FLAG_EXCLUSIVE标识
    	wq_entry->flags |= WQ_FLAG_EXCLUSIVE;
    	spin_lock_irqsave(&wq_head->lock, flags);
    	__add_wait_queue_entry_tail(wq_head, wq_entry);
    	spin_unlock_irqrestore(&wq_head->lock, flags);
    }


惊群参考 `这里 <http://wangxuemin.github.io/2016/01/25/Epoll%20%E6%96%B0%E5%A2%9E%20EPOLLEXCLUSIVE%20%E9%80%89%E9%A1%B9%E8%A7%A3%E5%86%B3%E4%BA%86%E6%96%B0%E5%BB%BA%E8%BF%9E%E6%8E%A5%E7%9A%84%E2%80%99%E6%83%8A%E7%BE%A4%E2%80%98%E9%97%AE%E9%A2%98/>`_


结构图示为:

.. code-block:: python

    '''
    
      (具体例子)sock结构 -+-----> sk_wq -+-->wait(wait_queue_head_t) -----> ... ------>
                                                                                 |
                                              |                                  | wait插入到poll_wait的尾部
                                              |whead指向具体的                   |
                                              |wait_queue头                      |
                                                                                 |
       pwq(eppoll_entry) -+----------------> whead                               |
                          |                                                      |
                          +-----> wait(wait_queue_entry_t) ----------------------+ --> func(wait_queue_func_t) = ep_poll_callback
                          |
                          +-----> llink ----------------------------
                          |                                        |llink插入到epitem的pwq_list
                          +-----> base(ep_item)                    |
                                                                   |的尾部
                                    |base指向epitem                |
                                    |                              |
                                                                   |
                                  epitem -+----> pwqlist -> ... ->
            
            
            
    '''


**所以, 每当file受信, 唤醒file对应的wait_queue中的wait_queue_entry, 调唤醒的操作是调用wait_queue_entry的回调, epoll中, 回调是ep_poll_callback**

**所以整体的epoll_insert就是查找fd, 然后操作各种wait_queue, 然后判断当前fd是否受信, 受信就加入到就绪列表中**

epoll->poll_wait
===================

epoll中除了wq这个wait_queue, 还有一个poll_wait的wait_queue.

在ep_item_poll函数中, 如果传入的epi对应的file是epoll对象, 那么就会把wait_queue_entry加入到epoll自己的poll_wait中, 那么当epoll中有

event受信的时候, 会唤醒poll_wait中的wait_queue_entry.

**其实这个poll_wait属性, 可以就类比于socket中的wait了**


.. code-block:: c

    static unsigned int ep_item_poll(struct epitem *epi, poll_table *pt, int depth)
    {
    	struct eventpoll *ep;
    	bool locked;
    
    	pt->_key = epi->event.events;
    	if (!is_file_epoll(epi->ffd.file))
    		return epi->ffd.file->f_op->poll(epi->ffd.file, pt) &
    		       epi->event.events;
    
        // 这里!!!!如果我们insert进来的file也是一个epoll对象的话
        // 走到poll_wait, 也就是ep_ptable_queue_proc中的whead就是
        // epi中file指向的另外一个epoll对象的poll_wait这个wait_queue
    	ep = epi->ffd.file->private_data;
    	poll_wait(epi->ffd.file, &ep->poll_wait, pt);
    	locked = pt && (pt->_qproc == ep_ptable_queue_proc);
    
    	return ep_scan_ready_list(epi->ffd.file->private_data,
    				  ep_read_events_proc, &depth, depth,
    				  locked) & epi->event.events;
    }

用ep_item_poll中if之后的代码, 和epoll自己的poll实现的代码对比, 其实一样

**所以ep_item_poll中if后面的代码就是执行了epoll自己的poll实现了**

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L923

.. code-block:: c

    static unsigned int ep_eventpoll_poll(struct file *file, poll_table *wait)
    {
    	struct eventpoll *ep = file->private_data;
    	int depth = 0;
    
    	/* Insert inside our poll wait queue */
    	poll_wait(file, &ep->poll_wait, wait);
    
    	/*
    	 * Proceed to find out if wanted events are really available inside
    	 * the ready list.
    	 */
    	return ep_scan_ready_list(ep, ep_read_events_proc,
    				  &depth, depth, false);
    }


受信回调
============

用socket作为例子


在socket中, 当socket可读的时候, 会调用到sock_def_readable, 而sock_def_readable会去唤醒

wait_queue中的wait_queue_entry, 也就是调用wait_queue_entry的回调, 这里回调是之前设置的ep_poll_callback

https://elixir.bootlin.com/linux/v4.15/source/net/core/sock.c#L2620

.. code-block:: c

    static void sock_def_readable(struct sock *sk)
    {
    	struct socket_wq *wq;
    
    	rcu_read_lock();
        // 拿到wait_queue
    	wq = rcu_dereference(sk->sk_wq);
        // 这个skwq_has_sleeper的判断是判断wa是否为空
        // 不为空就是真
    	if (skwq_has_sleeper(wq))
                // 去处理wq->wait
    		wake_up_interruptible_sync_poll(&wq->wait, POLLIN | POLLPRI |
    						POLLRDNORM | POLLRDBAND);
    	sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
    	rcu_read_unlock();
    }

wake_up_interruptible_sync_poll定义为

https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L215

.. code-block:: c

    #define wake_up_interruptible_sync_poll(x, m)					\
         __wake_up_sync_key((x), TASK_INTERRUPTIBLE, 1, (void *) (m))


注意下TASK_INTERRUPTIBLE这个进程状态的要求


__wake_up_common
===================

上面的__wake_up_sync_key会调用到__wake_up_common这个函数, 这个函数是基础的wake_up处理, 并且

__wake_up_sync_key传给__wake_up_common中的wake_flags一般是WF_SYNC

关于wakeup的flags

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/sched.h#L1375

.. code-block:: c

    /*
     * wake flags
     */
    #define WF_SYNC		0x01		/* waker goes to sleep after wakeup */
    #define WF_FORK		0x02		/* child wakeup after fork */
    #define WF_MIGRATED	        0x4		/* internal use, task got migrated */

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L192

.. code-block:: c

    void __wake_up_sync_key(struct wait_queue_head *wq_head, unsigned int mode,
    			int nr_exclusive, void *key)
    {
    	int wake_flags = 1; /* XXX WF_SYNC */
    
    	if (unlikely(!wq_head))
    		return;
    
        // 如果nr_exclusive不等于1, 那么wake_flags则是0
        // 0貌似没有定义
    	if (unlikely(nr_exclusive != 1))
    		wake_flags = 0;
    
    	__wake_up_common_lock(wq_head, mode, nr_exclusive, wake_flags, key);
    }

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L72

.. code-block:: c

    /*
     * The core wakeup function. Non-exclusive wakeups (nr_exclusive == 0) just
     * wake everything up. If it's an exclusive wakeup (nr_exclusive == small +ve
     * number) then we wake all the non-exclusive tasks and one exclusive task.
     *
     * There are circumstances in which we can try to wake a task which has already
     * started to run but is not in state TASK_RUNNING. try_to_wake_up() returns
     * zero in this (rare) case, and we handle it by continuing to scan the queue.
     */
    static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
    			int nr_exclusive, int wake_flags, void *key,
    			wait_queue_entry_t *bookmark)
    {
    	wait_queue_entry_t *curr, *next;
    	int cnt = 0;
    
    	if (bookmark && (bookmark->flags & WQ_FLAG_BOOKMARK)) {
    		curr = list_next_entry(bookmark, entry);
    
    		list_del(&bookmark->entry);
    		bookmark->flags = 0;
    	} else
    		curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry);
    
    	if (&curr->entry == &wq_head->head)
    		return nr_exclusive;
    
    	list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
    		unsigned flags = curr->flags;
    		int ret;
    
    		if (flags & WQ_FLAG_BOOKMARK)
    			continue;
    
                // 这里就是调用wait_queue_entry的回调的地方了!!!!
    		ret = curr->func(curr, mode, wake_flags, key);
    		if (ret < 0)
    			break;
    		if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
    			break;
    
    		if (bookmark && (++cnt > WAITQUEUE_WALK_BREAK_CNT) &&
    				(&next->entry != &wq_head->head)) {
    			bookmark->flags = WQ_FLAG_BOOKMARK;
    			list_add_tail(&bookmark->entry, &next->entry);
    			break;
    		}
    	}
    	return nr_exclusive;
    }

从注释上看, 参数nr_exclusive为0, 则是唤醒所有的进程, 而nr_exclusive大于0, 则是只会唤醒一个的exclusive模式的进程, 和所有的非exclusive模式的进程

*If it's an exclusive wakeup (nr_exclusive == small +venumber) then we wake all the non-exclusive tasks and one exclusive task.*

**而epoll中, ep_insert的时候, 都把wait_queue_entry设置上了exclusive标识(WQ_FLAG_EXCLUSIVE).**

ep_poll_callback
====================

这个是对应的file受信之后, 调用的回调, 这个是在ep_insert的时候调用的poll_wait函数中, 调用的ep_ptable_queue_proc中设置的

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L1114

.. code-block:: c

    static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
    {
    	int pwake = 0;
    	unsigned long flags;
    	struct epitem *epi = ep_item_from_wait(wait);
    	struct eventpoll *ep = epi->ep;
    	int ewake = 0;
    
    	spin_lock_irqsave(&ep->lock, flags);
    
    	ep_set_busy_poll_napi_id(epi);
    
    	/*
    	 * If the event mask does not contain any poll(2) event, we consider the
    	 * descriptor to be disabled. This condition is likely the effect of the
    	 * EPOLLONESHOT bit that disables the descriptor when an event is received,
    	 * until the next EPOLL_CTL_MOD will be issued.
    	 */
    	if (!(epi->event.events & ~EP_PRIVATE_BITS))
    		goto out_unlock;
    
    	/*
    	 * Check the events coming with the callback. At this stage, not
    	 * every device reports the events in the "key" parameter of the
    	 * callback. We need to be able to handle both cases here, hence the
    	 * test for "key" != NULL before the event match test.
    	 */
        // 如果发生的时间不是epi所关心的, 那么不唤醒
    	if (key && !((unsigned long) key & epi->event.events))
    		goto out_unlock;
    
    	/*
    	 * If we are transferring events to userspace, we can hold no locks
    	 * (because we're accessing user memory, and because of linux f_op->poll()
    	 * semantics). All the events that happen during that period of time are
    	 * chained in ep->ovflist and requeued later on.
    	 */
        // ovflist的作用, 下面说
    	if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
               // 如果需要把epi加入到ovflist的话
               // 那么直接跑到out_unlock代码块, 而不走下面的
               // 加入就绪链表的过程
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
    
        // 如果不需要把epi加入到ovflist的话
        // 把epi加入到就绪链表
    	/* If this file is already in the ready list we exit soon */
    	if (!ep_is_linked(&epi->rdllink)) {
    		list_add_tail(&epi->rdllink, &ep->rdllist);
    		ep_pm_stay_awake_rcu(epi);
    	}
    
    	/*
    	 * Wake up ( if active ) both the eventpoll wait list and the ->poll()
    	 * wait list.
    	 */
        // 这里的判断是ep->wq是否为空, 是空的话表示没有人监听, 就不唤醒了
        // 如果不为空, 则唤醒
    	if (waitqueue_active(&ep->wq)) {
    		if ((epi->event.events & EPOLLEXCLUSIVE) &&
    					!((unsigned long)key & POLLFREE)) {
    			switch ((unsigned long)key & EPOLLINOUT_BITS) {
    			case POLLIN:
    				if (epi->event.events & POLLIN)
    					ewake = 1;
    				break;
    			case POLLOUT:
    				if (epi->event.events & POLLOUT)
    					ewake = 1;
    				break;
    			case 0:
    				ewake = 1;
    				break;
    			}
    		}
                // 唤醒ep->wq中的进程
    		wake_up_locked(&ep->wq);
    	}
        // 唤醒自己的poll_wait
    	if (waitqueue_active(&ep->poll_wait))
    		pwake++;
    
    out_unlock:
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	/* We have to call this outside the lock */
    	if (pwake)
                // 这个唤醒的是自己的poll_wait
    		ep_poll_safewake(&ep->poll_wait);
    
    	if (!(epi->event.events & EPOLLEXCLUSIVE))
    		ewake = 1;
    
    	if ((unsigned long)key & POLLFREE) {
    		/*
    		 * If we race with ep_remove_wait_queue() it can miss
    		 * ->whead = NULL and do another remove_wait_queue() after
    		 * us, so we can't use __remove_wait_queue().
    		 */
    		list_del_init(&wait->entry);
    		/*
    		 * ->whead != NULL protects us from the race with ep_free()
    		 * or ep_remove(), ep_remove_wait_queue() takes whead->lock
    		 * held by the caller. Once we nullify it, nothing protects
    		 * ep/epi or even wait.
    		 */
    		smp_store_release(&ep_pwq_from_wait(wait)->whead, NULL);
    	}
    
    	return ewake;
    }


**ovflist是一个无锁情况下, 为了性能所使用的一个临时链表.**

比如当前有事件发生, 但是同时epoll正在把rdllist中的event赋值到用户态, 那么此时rdlist应该是允许操作的, 同时为了性能, 遍历

rdlist的时候, 是不加锁的, 所以此时的event受信不能操作rdlist, 所以只好放到另一一个备用的链表中了.

当epoll复制数据到用户态之后, ovflist就会被置为EP_UNACTIVE_PTR, 然后把ovflist中的epi添加到rdllist中.


而唤醒的过程是wake_up_locked这个函数, 是唤醒epoll->wq这个wait_queue的, 而wake_up_locked最后还是会跑到之前说的

__wake_up_common, 也就是遍历wait_queue, 然后调用wait_queue_entry中的func函数.

而wq是在调用ep_poll（epoll_wait)的时候时候把当前进程加入到wq的, 看下面


所以, 插入一个fd的时候, 调用改fd对应的file的poll实现的作用是:

**调用poll_table的func, ep_ptable_queue_proc, 把一个wait_queue_entry加入到该file的wait_queue中, 并且这个wait_queue_entry的回调func是ep_poll_callback**

**而ep_poll_callback的作用是判断是: 1. 是否是感兴趣的时间发生 2. 插入ovflist还是rdllist 3. 是否需要唤醒epoll->wq中的进程**

ep_insert的时候, 每一个wait_queue_entry都是加入了WQ_FLAG_EXCLUSIVE标识, 所以只会有一个wait_queue_entry被唤醒, 但是, ep_poll_callback中唤醒

epoll->wq的时候, 是否是唤醒多个, 取决于加入epoll->wq时候的是否有WQ_FLAG_EXCLUSIVE了, 看下面



epoll_wait/epoll_poll
========================

epoll_wait是系统调用, 调用函数epoll_poll去sleep

https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L1736

.. code-block:: c

    static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
    		   int maxevents, long timeout)
    {
    	int res = 0, eavail, timed_out = 0;
    	unsigned long flags;
    	u64 slack = 0;
    	wait_queue_entry_t wait;
    	ktime_t expires, *to = NULL;
    
        // 下面是检查tiumeout
        // 如果有timeout, 那么计算绝对时间
        // 如果没有, 那么直接跑到check_events代码部分
    	if (timeout > 0) {
    		struct timespec64 end_time = ep_set_mstimeout(timeout);
    
    		slack = select_estimate_accuracy(&end_time);
                // to是绝对时间
    		to = &expires;
    		*to = timespec64_to_ktime(end_time);
    	} else if (timeout == 0) {
    		/*
    		 * Avoid the unnecessary trip to the wait queue loop, if the
    		 * caller specified a non blocking operation.
    		 */
    		timed_out = 1;
    		spin_lock_irqsave(&ep->lock, flags);
                // 直接跑到check_events代码部分
    		goto check_events;
    	}
    
    // 这里是无限循环等待中断
    fetch_events:
    
    	if (!ep_events_available(ep))
    		ep_busy_loop(ep, timed_out);
    
    	spin_lock_irqsave(&ep->lock, flags);
    
        // 如果没有可用的event, 那么继续
    	if (!ep_events_available(ep)) {
    		/*
    		 * Busy poll timed out.  Drop NAPI ID for now, we can add
    		 * it back in when we have moved a socket with a valid NAPI
    		 * ID onto the ready list.
    		 */
    		ep_reset_busy_poll_napi_id(ep);
    
    		/*
    		 * We don't have any available event to return to the caller.
    		 * We need to sleep here, and we will be wake up by
    		 * ep_poll_callback() when events will become available.
    		 */
                // 注意, 这里是新建了一个wait(wait_queue_entry_t wait)
                // 然后设置wait的private设置为当前进程, 然后回调是默认的default_wake_function
    		init_waitqueue_entry(&wait, current);
                // 把当前进程加入到epoll->wq这个wait_queue链表中
                // 并且是exclusive模式, 避免惊群问题
    		__add_wait_queue_exclusive(&ep->wq, &wait);
    
    		for (;;) {
    			/*
    			 * We don't want to sleep if the ep_poll_callback() sends us
    			 * a wakeup in between. That's why we set the task state
    			 * to TASK_INTERRUPTIBLE before doing the checks.
    			 */
                        // 设置当前进程状态是可中断状态
                        // 这样sleep的时候可以被撞断唤醒
    			set_current_state(TASK_INTERRUPTIBLE);
    			/*
    			 * Always short-circuit for fatal signals to allow
    			 * threads to make a timely exit without the chance of
    			 * finding more events available and fetching
    			 * repeatedly.
    			 */
                        // 下面是检查进程的状态是否是被中断了
                        // 是的话break出循环
                        // 这里是说如果fd的poll调用在我们sleep之前, 已经发中断了
                        // 那么直接不用sleep了
    			if (fatal_signal_pending(current)) {
    				res = -EINTR;
    				break;
    			}
    			if (ep_events_available(ep) || timed_out)
    				break;
    			if (signal_pending(current)) {
    				res = -EINTR;
    				break;
    			}
    
    			spin_unlock_irqrestore(&ep->lock, flags);
                        // schedule_hrtimeout_range这个就是sleep until timeout了
    			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
    				timed_out = 1;
    
    			spin_lock_irqsave(&ep->lock, flags);
    		}
    
                // 跳出了循环
                // 要么被中断, 要么timeout了
    		__remove_wait_queue(&ep->wq, &wait);
    		__set_current_state(TASK_RUNNING);
    	}
    check_events:
    	/* Is it worth to try to dig for events ? */
        // 再次检查是否有可用的event
    	eavail = ep_events_available(ep);
    
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	/*
    	 * Try to transfer events to user space. In case we get 0 events and
    	 * there's still timeout left over, we go trying again in search of
    	 * more luck.
    	 */
        // 这里的ep_send_events就是把就绪列表中的event发送到用户态的缓冲区
    	if (!res && eavail &&
    	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
    		goto fetch_events;
    
    	return res;
    }


1. 把当前进程组成一个wait_queue_entry结构, 其func是default_wake_function, 然后被加入到当前epoll结构的wq(wait_queue_head_t)中, 这样有fd受信的时候, 会唤醒wq中的进程

2. schedule_hrtimeout_range是sleep until timeout的作用, 如果进程的状态被设置为TASK_UNINTERRUPTIBLE, 则不会被中断唤醒，如果TASK_INTERRUPTIBLE, 则收到中断, 那么也会被唤醒

3. ep_send_events会把对应的的就绪event发送到用户态缓冲区.

4. 把当前进程加入到epoll的wq这个wait_queue中, 并且是exclusive模式, 避免惊群问题.

5. ep_events_available这个函数是判断epoll是否有可用的event. 两者其中一个为真就是真: 1. 就绪列表是否不为空, 2. ovflist是否不是EP_UNACTIVE_PTR.

.. code-block:: c

    static inline int ep_events_available(struct eventpoll *ep)
    {
    	return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
    }


当前进程被加入到epoll->wq, 其中的wait_queue_entry的func属性被设置为default_wake_function, 而

之前看到的, ep_poll_callback会调用__wake_up_common去处理epoll->wq中的wait_queue_entry, 调用func, 所以

就是, 当一个event受信, 会调用default_wake_function

wait_up_locked
================

wake_up_locked是去唤醒指定的wait_queue上的wait_queue_entry

https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L198

.. code-block:: c

   #define wake_up_locked(x)		__wake_up_locked((x), TASK_NORMAL, 1)

所以就是调用__wake_up_locked, 传入的参数第二个是TASK_NORMAL, 第三个是1

而TASK_NORMAL则包含了TASK_INTERRUPTIBLE和TASK_UNINTERRUPTIBLE

.. code-block:: c

    #define TASK_NORMAL			(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)


而__wake_up_locked则是调用__wake_up_common, 并且把第三个参数作为nr_exclusive传给__wake_up_common

所以wake_up_locked也是只唤醒一个线程


default_wake_function
=========================

唤醒ep->wq上的wait_queue_entry的时候, 回调是default_wake_function

default_wake_function是调用try_to_wake_up这个函数, 这个函数是通用的唤醒程序的调用

所以理解上, 理解为把程序唤醒就好了.

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3625

.. code-block:: c

    int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags,
    			  void *key)
    {
    	return try_to_wake_up(curr->private, mode, wake_flags);
    }


唤醒并复制event到用户态
===========================


在之前的ep_poll函数中, 被唤醒的时候, 在check_events代码块, 还会去检查一次epoll中的就绪链表, 原因是防止fetch之后, 事件又变为不受信状态了

*Why does epoll check the event again in here? It does this to ensure the user registered event(s) are still available. Think about a scenario that a file descriptor got added into the ready list for EPOLLOUT while the user program writes to it. After the user program finishes writing, the file descriptor might no longer be available for writing anymore and epoll has to handle this case correctly otherwise the user will receive EPOLLOUT while the write operation will block.*

--- 参考自: https://idndx.com/2015/07/08/the-implementation-of-epoll-4/


然后调用ep_send_events去复制受信的event

ep_send_events
==================

将ep_send_events_proc传入给ep_scan_ready_list

.. code-block:: c

    static int ep_send_events(struct eventpoll *ep,
    			  struct epoll_event __user *events, int maxevents)
    {
    	struct ep_send_events_data esed;
    
    	esed.maxevents = maxevents;
    	esed.events = events;
    
    	return ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
    }

ep_scan_ready_list
======================

ep_scan_ready_list作用是:

1. 把ovflist设置不为EP_UNACTIVE_PTR状态, 这样是保护就绪链表的. 因为如果遍历就绪链表的时候, 同时有event受信, 那么为了不污染就绪链表, 受信的event会查看ovflist

   是否是EP_UNACTIVE_PTR, 如果不是, 那么不操作就绪链表而是暂时添加到ovflist链表中.
   
2. 但是对就绪链表的采取什么操作, 可以通过传入函数来指定.

3. 传入的ep_send_events_proc就是负责拷贝到用户态操作

4. 赋拷贝到用户态的时候同时又有受信发生, 再次唤醒

看注释:

.. code-block:: c

    /**
     * ep_scan_ready_list - Scans the ready list in a way that makes possible for
     *                      the scan code, to call f_op->poll(). Also allows for
     *                      O(NumReady) performance.
    */
    static int ep_scan_ready_list(struct eventpoll *ep,
    			      int (*sproc)(struct eventpoll *,
    					   struct list_head *, void *),
    			      void *priv, int depth, bool ep_locked)
    {
    	int error, pwake = 0;
    	unsigned long flags;
    	struct epitem *epi, *nepi;
    	LIST_HEAD(txlist);
    
    	/*
    	 * We need to lock this because we could be hit by
    	 * eventpoll_release_file() and epoll_ctl().
    	 */
    
    	if (!ep_locked)
    		mutex_lock_nested(&ep->mtx, depth);
    
    	/*
    	 * Steal the ready list, and re-init the original one to the
    	 * empty list. Also, set ep->ovflist to NULL so that events
    	 * happening while looping w/out locks, are not lost. We cannot
    	 * have the poll callback to queue directly on ep->rdllist,
    	 * because we want the "sproc" callback to be able to do it
    	 * in a lockless way.
    	 */
    	spin_lock_irqsave(&ep->lock, flags);
        // 复制一份就绪链表
    	list_splice_init(&ep->rdllist, &txlist);
        // 遍历的时候, 设置ovflist状态
        // 保护就绪链表
    	ep->ovflist = NULL;
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	/*
    	 * Now call the callback function.
    	 */
        // 调用传入的的操作函数
    	error = (*sproc)(ep, &txlist, priv);
    
    	spin_lock_irqsave(&ep->lock, flags);
    	/*
    	 * During the time we spent inside the "sproc" callback, some
    	 * other events might have been queued by the poll callback.
    	 * We re-insert them inside the main ready-list here.
    	 */
        // 遍历ovflist
        // 如果调用操作函数的时候, 同时有
        // event受信, 那么为了不漏掉这部分event, 需要
        // 把ovflist中的event加入到就绪链表
    	for (nepi = ep->ovflist; (epi = nepi) != NULL;
    	     nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
    		/*
    		 * We need to check if the item is already in the list.
    		 * During the "sproc" callback execution time, items are
    		 * queued into ->ovflist but the "txlist" might already
    		 * contain them, and the list_splice() below takes care of them.
    		 */
    		if (!ep_is_linked(&epi->rdllink)) {
    			list_add_tail(&epi->rdllink, &ep->rdllist);
    			ep_pm_stay_awake(epi);
    		}
    	}
    	/*
    	 * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after
    	 * releasing the lock, events will be queued in the normal way inside
    	 * ep->rdllist.
    	 */
        // 我们操作完就绪链表了
        // 可以开放就绪链表了
    	ep->ovflist = EP_UNACTIVE_PTR;
    
    	/*
    	 * Quickly re-inject items left on "txlist".
    	 */
    	list_splice(&txlist, &ep->rdllist);
    	__pm_relax(ep->ws);
    
        // 就绪链表不为空
    	if (!list_empty(&ep->rdllist)) {
    		/*
    		 * Wake up (if active) both the eventpoll wait list and
    		 * the ->poll() wait list (delayed after we release the lock).
    		 */
                // 唤醒进程!!!
    		if (waitqueue_active(&ep->wq))
    			wake_up_locked(&ep->wq);
    		if (waitqueue_active(&ep->poll_wait))
    			pwake++;
    	}
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	if (!ep_locked)
    		mutex_unlock(&ep->mtx);
    
    	/* We have to call this outside the lock */
    	if (pwake)
    		ep_poll_safewake(&ep->poll_wait);
    
    	return error;
    }

1. 复制一份就绪链表: list_splice_init(&ep->rdllist, &txlist);

2. 操作就绪链表之前, 设置ovflist: ep->ovflist = NULL, 此时ovflist不等于EP_UNACTIVE_PTR, 保护就绪链表

3. 调用传入的操作函数: error = (\*sproc)(ep, &txlist, priv);

4. 操作函数处理完之后, 为了不漏掉同时发生的event, 把ovflist上的event赋值到就绪链表

5. 设置ovflist为EP_UNACTIVE_PTR状态: ep->ovflist = EP_UNACTIVE_PTR, 开放就绪链表操作

6. 如果ovflist复制到就绪链表之后, 就绪链表不为空, 那么表示同时有event受信, 然后唤醒进程.


只会唤醒一个
===============

代码流程分两部分

这里会和惊群有点理解上的疑惑, 后面讲

这里先讲如何只唤醒wait_queue上其中一个wait_queue_entry

回调的exclusive
-------------------

之前的流程中, 在insert的时候, 调用目标file的poll实现, 是添加wait_queue_entry, 并且改wait_queue_entry的回调是ep_poll_callback.

作为例子, 假设socket变为可读状态, 那么sock_def_readable中调用wake_up_interruptible_sync_poll去唤醒自己的wait_queue.

其中wake_up_interruptible_sync_poll的定义是:

https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L215

.. code-block:: c

    #define wake_up_interruptible_sync_poll(x, m)					\
    	__wake_up_sync_key((x), TASK_INTERRUPTIBLE, 1, (void *) (m))
    

注意__wake_up_sync_key的第三个参数, 先看看__wake_up_sync_key的定义:

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L192

.. code-block:: c

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
    EXPORT_SYMBOL_GPL(__wake_up_sync_key);

__wake_up_sync_key第三个参数是nr_exclusive, 和唤醒多少个wait_queue_entry有关, 而__wake_up_common_lock会把nr_exclusive这个参数传入到

__wake_up_common中, 而__wake_up_common中, nr_exclusive的作用是:

*The core wakeup function. Non-exclusive wakeups (nr_exclusive == 0) just wake everything up.
If it's an exclusive wakeup (nr_exclusive == small +venumber) then we wake all the non-exclusive tasks and one exclusive task.*

也就是nr_exclusive如果是0, 那么会唤醒所有的wait_queue_entry, 如果大于0, **那么唤醒一个exclusive的wait_queue_entry和所有的非exclusive的wait_queue_entry**

https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L72

.. code-block:: c

    static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
    			int nr_exclusive, int wake_flags, void *key,
    			wait_queue_entry_t *bookmark)
    {
        // 拿到wait_queue_entry的flag
        unsigned flags = curr->flags;
       
        // 省略代码
    
        // 遍历wait_queue
        // 注意, 这里遍历的时候也会删除遍历到的元素的!!!!
    	list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
    		unsigned flags = curr->flags;
    		int ret;
    
    		if (flags & WQ_FLAG_BOOKMARK)
    			continue;
    
    		ret = curr->func(curr, mode, wake_flags, key);
    		if (ret < 0)
    			break;
                // 注意看这个判断
    		if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
    			break;
                    // 省略代码
             }
    
            // 省略代码
    
    }

从上面的代码中看到, 遍历到一个wait_queue_entry, 调用其func之后, 如果成功, 并且flag是WQ_FLAG_EXCLUSIVE, 并且nr_exclusive已经减少到0, 那么退出.

由于:

1. 我们在调用socket的poll实现的时候, 最后会调用到(tcp_poll -> poll_wait)ep_ptable_queue_proc中
   
   而该函数是调用add_wait_queue_exclusive把带有EPOLLEXCLUSIVE标志位的wait_queue_entry设置上WQ_FLAG_EXCLUSIVE标识的.

   EPOLLEXCLUSIVE需要手动传入, 比如调用epoll_ctl的时候传入

.. code-block:: c

    static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
    				 poll_table *pt)
    {
        if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
            // 省略一些代码

            // 注意这里的EPOLLEXCLUSIVE判断
            // 如果我们手动传入了EPOLLEXCLUSIVE标志位, 那么
            // 就为wait_queue_entry加入WQ_FLAG_EXCLUSIVE标志位
            if (epi->event.events & EPOLLEXCLUSIVE)
    	        add_wait_queue_exclusive(whead, &pwq->wait);
    	    else
    	        add_wait_queue(whead, &pwq->wait);
            }
            // 省略一些代码
    }

2. 目标的file的wait_queue_entry的func是ep_poll_callback, 其中调用的时候, 向__wake_up_common传入的nr_exclusive是1!!!

**所以socket受信的时候, 只会唤醒一个wait_queue_entry!!**

而在ep_poll_callback中, 会调用wake_up_locked去唤醒epoll->wq中的进程:

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/fs/eventpoll.c#L1114
    static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
    {
        
        if (waitqueue_active(&ep->wq)) {
            // 唤醒epoll->wq
            wake_up_locked(&ep->wq);
        }
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/wait.h#L198
    #define wake_up_locked(x)		__wake_up_locked((x), TASK_NORMAL, 1)
   
    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/wait.c#L156
    void __wake_up_locked(struct wait_queue_head *wq_head, unsigned int mode, int nr)
    {
    	__wake_up_common(wq_head, mode, nr, 0, NULL, NULL);
    }

**可以看到, 调用传入__wake_up_common的nr_exclusive参数也是1**, 所以__wake_up_common中的判断:

.. code-block:: c

    if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
        break


3. list_for_each_entry_safe_from遍历的时候, 会把遍历到的元素给删除掉!!!!

是否是只唤醒一个进程, 只需要看看flag是否是WQ_FLAG_EXCLUSIVE了.

所以, 即使只唤醒了一个wait_queue_entry, 在wait_queue_entry的回调中, 唤醒epoll->wq的时候还是可能会唤醒多个进程的, 取决于进程加入到epoll->wq时候的flag了


关于唤醒进程的exclusive
-------------------------

当调用ep_poll的时候, 把当前经常加入到epoll->wq的方式也是exclusive的:

.. code-block:: c

    static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
    		   int maxevents, long timeout)
    {
    
        init_waitqueue_entry(&wait, current);
        __add_wait_queue_exclusive(&ep->wq, &wait);
    
    
    }

其中wait_queue_entry的回调是default_wake_function, 并且其flag是WQ_FLAG_EXCLUSIVE, 关于default_wake_function, 也就是唤醒指定进程的操作了

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/sched/core.c#L3625
    int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags,
    			  void *key)
    {
        // 之前init_waitqueue_entry的时候, 已经把当前进程
        // 存储到private这个属性中了
    	return try_to_wake_up(curr->private, mode, wake_flags);
    }


**所以, WQ_FLAG_EXCLUSIVE和nr_exclusive, 两个参数指定了epoll的返回是exclusive的**

**注意, 这里的__add_wait_queue_exclusive是头插!!! wait_queue_entry同样是头插!!**

但是!!!
=========

及时在4.4的内核中, 由于ep_ptable_queue_proc中添加wait_queue_entry到目标file的wait_queue的时候, 没有

带上exclusive标识, 所以还是会惊群的. 下面是4.4内核的代码

.. code-block:: c

    https://elixir.bootlin.com/linux/v4.4/source/fs/eventpoll.c#L1088
    static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
    				 poll_table *pt)
    {
    	struct epitem *epi = ep_item_from_epqueue(pt);
    	struct eppoll_entry *pwq;
    
    	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
    		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
    		pwq->whead = whead;
    		pwq->base = epi;
                // 看这里!!!!
    		add_wait_queue(whead, &pwq->wait);
    		list_add_tail(&pwq->llink, &epi->pwqlist);
    		epi->nwait++;
    	} else {
    		/* We have to signal that an error occurred */
    		epi->nwait = -1;
    	}
    }

4.4中是add_wait_queue, 而4.15的话是判断一下EPOLLEXCLUSIVE, 然后调用add_wait_queue_exclusive的

ET和LT的区别
==================

ep_poll的时候, 给ep_scan_ready_list传入的函数是ep_send_events_proc

这个函数是复制数据到用户态的, 然后还有针对LE/ET模式的区别

简单来讲就是, 如果是LT模式, 则把event再次放入rdllist中

.. code-block:: c

    static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
    			       void *priv)
    {
    
        // 不是ET模式, 则再次放入rdllist中
        else if (!(epi->event.events & EPOLLET)) {
            list_add_tail(&epi->rdllink, &ep->rdllist);
        }
    
    }


所以

1. 如果你是LT模式的话, 你读了一部分数据, 不校验数据是否读完了, 然后继续ep_poll, 还是能被提醒说有数据可读的.

2. 在ET模式下, 如果你读了某fd的数据n大于你要读的大小m, 此时你读取的m, 但是还剩下n-m, 如果你不继续检验数据是否读完了

   那么你继续ep_poll的话, rdllist上没有该fd, 所以是不会拿到通知的.


多核惊群!!!!!!
=================

多个task去调用同一个socket的accept, 不会惊群, 这个问题是内核里面解决了, 代码没找到, 而epoll多核会惊群的

下面的例子环境是ubuntu 18(内核4.15), 主线程称为A, 子线程称为B

例子1
----------

多线程共享一个epoll对象进行read, 发生惊群

.. code-block:: python


    import threading
    import socket
    import selectors
    import select
    
    
    def read(name, fobj):
        data = fobj.recv(1)
        print('%s got data' % name, data)
        return
    
    ss = {}
    
    
    def child(stor, sock):
        print('child runing')
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock)
        return
    
    
    def main():
        accept_sock = socket.socket()
        accept_sock.bind(('0.0.0.0', 1993))
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN)
        th = threading.Thread(target=child, args=(stor, sock))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock)
        return
    
    
    if __name__ == '__main__':
        main()

输出是A线程一直读取1字节然后打印出来, 然后B也被唤醒, 然后B报了一个erron=11, 也就是EAGAIN

这是因为在LT模式下, 我们会把fd再次加入到就绪链表中, 然后在ep_scan_ready_list中, 会去再次唤醒监听在epoll上的线程

例子中就是B, 然后唤醒. 我们来看看源码, 回到ep_scan_ready_list和ep_send_events_proc中

先看看ep_send_events_proc如果处理LT模式的

.. code-block:: c

    static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
    			       void *priv)
    {
    
    	for (eventcnt = 0, uevent = esed->events;
    	     !list_empty(head) && eventcnt < esed->maxevents;) {
                // 拿到epi, 这里总是拿第一个
                // 下面会把遍历到的epi从rdllist中给删除掉
    	    epi = list_first_entry(head, struct epitem, rdllink);
    
    	    ws = ep_wakeup_source(epi);
    	    if (ws) {
    	    	if (ws->active)
    	    		__pm_stay_awake(ep->ws);
    	    	__pm_relax(ws);
    	    }
    
                // 把遍历到的epi从rdllist中删除
    	    list_del_init(&epi->rdllink);
    
    	    revents = ep_item_poll(epi, &pt, 1);
    
    	    /*
    	     * If the event mask intersect the caller-requested one,
    	     * deliver the event to userspace. Again, ep_scan_ready_list()
    	     * is holding "mtx", so no operations coming from userspace
    	     * can change the item.
    	     */
                // 注意, put_user就是把数据赋值到uevent
                // uevent = priv, 也就是上一层传入的, 返回给用户的数据结构
    	    if (revents) {
    	    	if (__put_user(revents, &uevent->events) ||
    	    	    __put_user(epi->event.data, &uevent->data)) {
    	    		list_add(&epi->rdllink, head);
    	    		ep_pm_stay_awake(epi);
    	    		return eventcnt ? eventcnt : -EFAULT;
    	    	}
    	    	eventcnt++;
    	    	uevent++;
    	    	if (epi->event.events & EPOLLONESHOT)
    	    		epi->event.events &= EP_PRIVATE_BITS;
                    // 这里!!!!!!
                    // 如果是LT模式, 那么直接再加入到rdllist中
    	    	else if (!(epi->event.events & EPOLLET)) {
    	    		/*
    	    		 * If this file has been added with Level
    	    		 * Trigger mode, we need to insert back inside
    	    		 * the ready list, so that the next call to
    	    		 * epoll_wait() will check again the events
    	    		 * availability. At this point, no one can insert
    	    		 * into ep->rdllist besides us. The epoll_ctl()
    	    		 * callers are locked out by
    	    		 * ep_scan_ready_list() holding "mtx" and the
    	    		 * poll callback will queue them in ep->ovflist.
    	    		 */
    	    		list_add_tail(&epi->rdllink, &ep->rdllist);
    	    		ep_pm_stay_awake(epi);
    	    	}
    	    }
    	}
    
    
    }


所以, 如果是LT模式, 一个fd会被再次加入到rdllist中, 在我们的例子中, 也就是复制数据A的时候, ep_wait没有返回给A

之前, fd将被再次加入到rdllist!!! 然后我们看看ep_scan_ready_list


.. code-block:: c

    static int ep_scan_ready_list(struct eventpoll *ep,
    			      int (*sproc)(struct eventpoll *,
    					   struct list_head *, void *),
    			      void *priv, int depth, bool ep_locked)
    {
    
        // 这里, 传入的sproc就是ep_send_events_proc
        error = (*sproc)(ep, &txlist, priv);
        
        // ep_send_events_proc返回了, 也就是
        // 复制了数据到用户空间, 也就是传入的priv
        // priv这个结构是上一层传入的
        ep->ovflist = EP_UNACTIVE_PTR;
        
        /*
         * Quickly re-inject items left on "txlist".
         */
        // 我们把txlist加入到rdllist
        list_splice(&txlist, &ep->rdllist);
        
        __pm_relax(ep->ws);
        
        // 如果rdllist不为空, 那么继续去唤醒task
        if (!list_empty(&ep->rdllist)) {
        	/*
        	 * Wake up (if active) both the eventpoll wait list and
        	 * the ->poll() wait list (delayed after we release the lock).
        	 */
        	if (waitqueue_active(&ep->wq))
        		wake_up_locked(&ep->wq);
        	if (waitqueue_active(&ep->poll_wait))
        		pwake++;
        }
    
    }

其中会判断rdllist是否为空, 而我们例子中不为空, 因为线程B依然在监听, 我们唤醒A的时候, 因为exclusive标志位的关系

只会唤醒线程A, 这个没问题, 然后LT下, 唤醒A的同时, ep_scan_ready_list会判断rdllist中依然有fd, 那么, 再次去唤醒

线程B, 所以接下来的流程就是, A先读取数据, B之后也被唤醒, 然后B又读取数据, 如果A已经把数据读取完了

那么fd缓存区没有数据, 那么B就遇到error=11, EAGAIN, 如果A没读取完, 那么剩下的数据就被B读取到

**所以, 情况就是A, B线程都会读取到数据, 然后其中有一个会出现EAGAIN**

例子2
----------

多线程使用同一个epoll对象监听同一个fd, 但是epoll_ctl的时候, 加入EPOLLEXCLUSIVE标志位

.. code-block:: python

    import threading
    import socket
    import selectors
    import select
    
    
    def read(name, fobj):
        data = fobj.recv(1)
        print('%s got data' % name, data)
        return
    
    ss = {}
    
    
    def child(stor, sock):
        print('child runing')
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock)
        return
    
    
    def main():
        accept_sock = socket.socket()
        accept_sock.bind(('0.0.0.0', 1993))
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLEXCLUSIVE)
        th = threading.Thread(target=child, args=(stor, sock))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock)
        return
    
    
    if __name__ == '__main__':
        main()

同样, 结果和例子1一样

例子3
---------

多线程使用不同的epoll对象去监听同一个fd对象, 加入EPOLLEXCLUSIVE标志位

.. code-block:: python

    import threading
    import socket
    import selectors
    import select
    
    
    def read(name, fobj):
        data = fobj.recv(1)
        print('%s got data' % name, data)
        return
    
    ss = {}
    
    
    def child(sock):
        print('child runing')
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLEXCLUSIVE)
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock)
        return
    
    
    def main():
        accept_sock = socket.socket()
        accept_sock.bind(('0.0.0.0', 1993))
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLEXCLUSIVE)
        th = threading.Thread(target=child, args=(sock,))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock)
        return
    
    
    if __name__ == '__main__':
        main()

此时只有一个线程被唤醒, 并且不会出现EAGAIN错误

但是, 如果在read中, recv后面sleep一下, 那么依然会出现B线程被唤醒然后数据交错

这是因为A线程没读取完数据的时候, 此时有数据, 此时fd的wait_queue上有B, 虽然带上了WQ_FLAG_EXCLUSIVE, 但是

还是会唤醒一个线程, 也就是B, 此时B也开始接收数据了

**所以, LT的EPOLLEXCLUSIVE标志位只是说能阻止同时地去获取数据, 比如A, B同时被唤醒, 而能阻止说

A先唤醒, 但是A没接收完数据的时候, 比如A一次接收1024字节, 但是传入1027字节, 然后再次传入3字节, 那么B还是被唤醒, B收到3+3=6字节!!!**


小结
===========

例子1就是一般性的惊吓, 然后我们可以和参考6中的例子比对

参考6中说使用EPOLLEXCLUSIVE标志位也不能阻止read的惊群, 但是参考6中说的情况是例子2

也就是多个线程同一个epoll监听一个fd, 不管加不加入EPOLLEXCLUSIVE, 都会发生惊群的, 这是因为

fd如果是可读的, 那么LT下总是重新把fd加入到就绪队列, 然后ep_scan_ready_list中判断是否还有监听的task

有则唤醒, 如果是同一个epoll对象的话, 明显, 线程B会被唤醒, 所以B也去读取数据


如果是不同的epoll对象的话, EPOLLEXCLUSIVE就起作用了

EPOLLEXCLUSIVE的作用其实是说往wait_queue_entry加入WQ_FLAG_EXCLUSIVE标志位, 那么也就是

说, fd上有两个wait_queue_entry, 每一个都表示epoll对象, 然后wait_queue_entry上带有WQ_FLAG_EXCLUSIVE标志

那么fd唤醒的时候, 只会唤醒一个epoll对象, 比如A, 然后A读取, 发现自己是LT模式, 然后把fd重新加入到自己的就绪链表,

那么A继续epoll_wait的时候, 又立马返回去读取数据, 同时把新建一个wait_queue_entry对象加入到fd的wait_queue中, 注意, 此时是头插, 所以就选A

读取完数据了, 之后又有数据进来, 那么很可能就是A, 因为遍历wait_queue是顺序遍历, 而插入wait_queue是头插法

**所以, EPOLLEXCLUSIVE是有用的, 只是记得要分开epoll对象, 因为EPOLLEXCLUSIVE是使用了wait_queue上的WQ_FLAG_EXCLUSIVE标志位**

EPOLLEXCLUSIVE的作用在参考[9]_中也有说明

**但是, 多个epoll对象使用EPOLLEXCLUSIVE标志也不能完全阻止惊群, 参考[6]_也是建议说使用ET模式加上EPOLLONESHOT标志位**


ET的read
============

ET的read一般是正常的, 但是在参考[9]_中说也会出现A, B分别拿到数据的情况, 其实是这样的

A, B使用用一个epoll对象, 然后A拿到数据之后做处理, 此时又收到了数据, 那么B被唤醒

下面的例子中, A每收到一个字节, 就sleep(1), 在A没有接收完毕的时候, 又发送数据, 那么B被唤醒

此时出现数据交错

.. code-block:: python

    import threading
    import errno
    import socket
    import selectors
    import select
    import time
    
    
    def read(name, fobj):
        data = ''
        while True:
            try:
                tmp = fobj.recv(1)
                print('recv tmp', tmp)
                time.sleep(1)
                if tmp:
                    data += tmp.decode()
                else:
                    break
            except BlockingIOError as e:
                print(e)
                break
        print('%s got ' % name, data)
        return
    
    ss = {}
    
    
    def child(stor, sock):
        print('child runing')
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock)
        return
    
    
    def main():
        accept_sock = socket.socket()
        while True:
            try:
                accept_sock.bind(('0.0.0.0', 1993))
            except OSError:
                time.sleep(1)
                continue
            break
        print('accept done')
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLET)
        th = threading.Thread(target=child, args=(stor, sock))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock)
        return
    
    
    if __name__ == '__main__':
        main()

然后我们使用EPOLLONESHOT之后, 就正常了, 只有一个线程被唤醒去获取数据

也就是A被唤醒去读取数据, 在A进行sleep中, 有数据进来, 那么B不会被唤醒的!!!

下面的例子就是加入了EPOLLONESHOT

.. code-block:: python

    import threading
    import errno
    import socket
    import selectors
    import select
    import time
    
    
    def read(name, fobj):
        data = ''
        while True:
            try:
                tmp = fobj.recv(1)
                print('recv tmp', tmp)
                time.sleep(1)
                if tmp:
                    data += tmp.decode()
                else:
                    break
            except BlockingIOError as e:
                print(e)
                break
        print('%s got ' % name, data)
        return
    
    ss = {}
    
    
    def child(stor, sock):
        print('child runing')
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock)
        return
    
    
    def main():
        accept_sock = socket.socket()
        while True:
            try:
                accept_sock.bind(('0.0.0.0', 1993))
            except OSError:
                time.sleep(1)
                continue
            break
        print('accept done')
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLET|select.EPOLLONESHOT)
        th = threading.Thread(target=child, args=(stor, sock))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock)
        return
    
    
    if __name__ == '__main__':
        main()


但是, 有一个问题, 就是当A接收完数据, 碰到EAGAIN之后, 继续epoll_wait, 然后捏, 你再次发送数据的时候, 没有

线程被唤醒了, 也就是A, B都不会被唤醒了!!!!!

这个时候需要我们手动调用epoll_ctl(EPOLL_CTL_MOD)去重置fd

.. code-block:: python

    import threading
    import errno
    import socket
    import selectors
    import select
    import time
    
    
    def read(name, fobj, ep):
        data = ''
        while True:
            try:
                tmp = fobj.recv(1)
                print('recv tmp', tmp)
                time.sleep(1)
                if tmp:
                    data += tmp.decode()
                else:
                    break
            except BlockingIOError as e:
                print(e)
                print('modify')
                # 看这里, 需要modify!!!!!!!!!!
                # 注意, 这里除了是EPOLLIN, 还必须带上EPOLLONSHOT!!
                ep.modify(fobj.fileno(), select.EPOLLIN|select.EPOLLONESHOT)
                break
        print('%s got ' % name, data)
        return
    
    ss = {}
    
    
    def child(stor, sock):
        print('child runing')
        while True:
            events = stor.poll()
            print('---------------child wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('---------child', sock, stor)
        return
    
    
    def main():
        accept_sock = socket.socket()
        while True:
            try:
                accept_sock.bind(('0.0.0.0', 1993))
            except OSError:
                time.sleep(1)
                continue
            break
        print('accept done')
        accept_sock.listen(100)
        sock = accept_sock.accept()[0]
        sock.setblocking(False)
        stor = select.epoll()
        stor.register(sock, select.EPOLLIN|select.EPOLLET|select.EPOLLONESHOT)
        th = threading.Thread(target=child, args=(stor, sock))
        th.start()
        while True:
            events = stor.poll()
            print('+++++++++++++++master wakeup', events)
            for k, v in events:
                if v == select.EPOLLIN:
                    read('+++++++++++master', sock, stor)
        return
    
    
    if __name__ == '__main__':
        main()


注意的是, 我们modify的时候, 记得加上EPOLLONSHOT, 否则下次接收数据又会出现两个线程同时接收数据

出现数据交错的情况

accept惊群
=============

accpet在LT模式下和read一样, 同一个epoll对象的话, 加不加EPOLLEXCLUSIVE都不能阻止惊群, 只有多个epoll对象

并且设置EPOLLEXCLUSIVE标志位才可能阻止惊群

下面是参考[6]_中例子的理解, 假设条件是一开始有1个连接进来, 在A中有:

.. code-block:: python
   
   try:
       while True:
           accept
   except BlockingIOError:
       pass

那么A进行一次accept之后, fd此时是non readable状态, 那么此时A再次epoll_wait之前, 有一个连接进来, 那么此时只有B一个

线程等待, 然后epoll唤醒B, 然后B进行accept, 然后因为A必须等到EAGAIN, 也就是BlockingIOError, 才会再次调动epoll_wait

所以, A继续accept, 而B因为也被唤醒了, 此时B进行accept, 那么B就遇到EAGAIN, 也就是B的唤醒是没必要的

然后在ET下, 参考[6]_中说accpet还会出现load balance的饥饿现象, **假设条件是一开始有2个连接!!!!**

在A中是

.. code-block:: python
   
   try:
       while True:
           accept
   except BlockingIOError:
       pass

因为A进行accept, 此时fd还是readable状态, 那么此时又有第3个连接进来, 因为fd保存readable状态, 不会唤醒线程B

然后假如一直有连接进来, 所有A一直都不会遇到BlockingIOError, 那么

而B线程一直都是在等待状态, 这样就达不到load balance了, 此时需要手动调用epoll_ctl(EPOLL_CTL_MOD, ...)

accept在LT的解决和read一样, 多个epoll对象加上EPOLLEXCLUSIVE

accpet在ET的解决和read一样, 同一个epoll对象的话, 手动调用epoll_ctl

EPOLLONSHOT
===================

这个是在ET模式下, 需要手动再调用epoll_ctl去重置fd, 否则后续的数据不会去唤醒任何线程

1. 当我们传入的event带上有EPOLLONESHOT的话, 唤醒的时候在ep_send_events_proc中

   不会加入到就绪链表中, 如果只是简单的ET模式, 也不会加入到就绪链表的


.. code-block:: c

    if (revents) {
    	if (__put_user(revents, &uevent->events) ||
    	    __put_user(epi->event.data, &uevent->data)) {
    		list_add(&epi->rdllink, head);
    		ep_pm_stay_awake(epi);
    		return eventcnt ? eventcnt : -EFAULT;
    	}
    	eventcnt++;
    	uevent++;
        // 这里设置上EP_PRIVATE_BITS
        // 并且不会走下面判断ET模式的代码块
    	if (epi->event.events & EPOLLONESHOT)
    		epi->event.events &= EP_PRIVATE_BITS;
    	else if (!(epi->event.events & EPOLLET)) {
    		/*
    		 * If this file has been added with Level
    		 * Trigger mode, we need to insert back inside
    		 * the ready list, so that the next call to
    		 * epoll_wait() will check again the events
    		 * availability. At this point, no one can insert
    		 * into ep->rdllist besides us. The epoll_ctl()
    		 * callers are locked out by
    		 * ep_scan_ready_list() holding "mtx" and the
    		 * poll callback will queue them in ep->ovflist.
    		 */
    		list_add_tail(&epi->rdllink, &ep->rdllist);
    		ep_pm_stay_awake(epi);
    	}
    }


    // 而EP_PRIVATE_BIT有
    #define EP_PRIVATE_BITS (EPOLLWAKEUP | EPOLLONESHOT | EPOLLET | EPOLLEXCLUSIVE)


EP_PRIVATE_BITS是一堆标志位的集合, 此时, 假设没有加上EPOLLONESHOT标志位, 

如果此时又有数据进来, 我们来看看ep_poll_callback

2. ep_poll_callback

.. code-block:: c

    static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
    {
    
        // 注意, 如果之前没带有EPOLLONESHOT的话
        // 那么这个判断是过不了的, 走到下面的代码
    	if (!(epi->event.events & ~EP_PRIVATE_BITS))
    		goto out_unlock;
    
        // 这里, wake_up_locked去唤醒
    	if (waitqueue_active(&ep->wq)) {
    	    if ((epi->event.events & EPOLLEXCLUSIVE) &&
    	    			!((unsigned long)key & POLLFREE)) {
    	    	switch ((unsigned long)key & EPOLLINOUT_BITS) {
    	    	case POLLIN:
    	    		if (epi->event.events & POLLIN)
    	    			ewake = 1;
    	    		break;
    	    	case POLLOUT:
    	    		if (epi->event.events & POLLOUT)
    	    			ewake = 1;
    	    		break;
    	    	case 0:
    	    		ewake = 1;
    	    		break;
    	    	}
    	    }
    	    wake_up_locked(&ep->wq);
    	}
    	if (waitqueue_active(&ep->poll_wait))
    		pwake++;
    
    }

不带EPOLLONESHOT, 然后, 此时又有数据进来, 比如A没读取完数据, 有数据进来, 此时会唤醒B, 这是ep_poll_callback中, 因为

判断语句 *epi->event.events & ~EP_PRIVATE_BITS* 为1, 那么*!(epi->event.events & ~EP_PRIVATE_BITS)*就是 !(1)=0, 那么就走到下面wake_up_locked

的唤醒过程, 如果之前带上了EPOLLONESHOT的话, 那么判断语句 *epi->event.events & ~EP_PRIVATE_BITS* 就是

*EP_PRIVATE_BITS & ~EP_PRIVATE_BITS* = 0, 所以那么*!(epi->event.events & ~EP_PRIVATE_BITS)*就是!(0) = 1

直接走out_unlock代码块, 不走唤醒流程, 所有A可以一直读取数据直到EAGAIN


3. 当我们手动调用epoll_ctl(EPOLL_CTL_MOD, EPOLLIN|EPOLLONESHOT)的时候, 调用modify去重置fd

   注意, 如果是EPOLLONESHOT的话, 和EPOLLEXCLUSIVE进行and(&)操作也是true, 所有走到ep_modify函数


.. code-block:: c

    SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
    		struct epoll_event __user *, event)
    {
    
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
    	case EPOLL_CTL_DEL:
    		if (epi)
    			error = ep_remove(ep, epi);
    		else
    			error = -ENOENT;
    		break;

        // 这里, 调用!!!ep_modify去重置fd
        // 虽然这里是判断EPOLLEXCLUSIVE, 但是我们知道刚刚
        // 我们把event添加上了EP_PRIVATE_BITS, 该标志位包含了
        // EPOLLEXCLUSIVE
        case EPOLL_CTL_MOD:
    		if (epi) {
    			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
    				epds.events |= POLLERR | POLLHUP;
    				error = ep_modify(ep, epi, &epds);
    			}
    		} else
    			error = -ENOENT;
    		break;
    	}
    
    }

3. 关于ep_modify, 注释上感觉就是说重置fd的功能

.. code-block:: c

    /*
     * Modify the interest event mask by dropping an event if the new mask
     * has a match in the current file status. Must be called with "mtx" held.
     */
    static int ep_modify(struct eventpoll *ep, struct epitem *epi, struct epoll_event *event)
    {
    
    
    }


这样, A读取完数据了进入wait状态, 重置fd的event为(EPOLLIN|EPOLLONESHOT), 而不是EP_PRIVATE_BITS

这样接下来再有数据进来的话, 又有线程可以被唤醒了, 因为此时在ep_poll_callback中判断语句

*!(epi->event.events & ~EP_PRIVATE_BITS)* 就为!(1) = 0, 那么就不会走判断语句下面的代码, 就直接走到唤醒的流程

EP_PRIVATE_BITS = 1<<31 | 1<<30 | 1<< 29 | 1<<28, 也就是

1111000000...000(一共32位), ~EP_PRIVATE_BITS = 0000111...1111(一共32位)


EPOLLONESHOT/EPOLLEXCLUSIVE
=====================================

EPOLLEXCLUSIVE根据参考[9]_的内核补丁的说明是: 多个epoll对象可以使用这个标志, 这个标志呢则会

使用wait_queue中的WQ_FLAG_EXCLUSIVE标志, 是得在fd上有多个epoll对象在监听的话, 只有一个epoll对象被唤醒

而EPOLLONESHOT是说在ET模式下, 只要该fd是处于EPOLLONESHOT状态, 那么不会被主动唤醒, 必须手动调用

epoll_ctl(EPOLL_CTL_MOD, EPOLLIN|EPOLLONESHOT)去重置fd的event, 这样后续的有数据的时候才会有线程被唤醒

nginx惊群解决
----------------

在worker去调用ep_wait的时候, 先抢一个互斥锁, 抢到才能调用ep_wait, 否则继续等待, 并且加入一个负载均衡

当一个worker抢锁的次数达到总次数的7/8的时候, 就不再抢锁, 给其他锁有抢锁的机会

比如A达到了7/8, 那么不抢锁了, 然后B达到7/8, 此时A又可以去抢锁了

