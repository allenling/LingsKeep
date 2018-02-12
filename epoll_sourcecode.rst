epoll
=========

内核版本v4.15

参考1: https://idndx.com/2014/09/01/the-implementation-of-epoll-1/, 这个系列有4章

参考2: http://blog.csdn.net/kai8wei/article/details/51233494

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



epoll在大多数情况是空闲的时候, 比起select快很多, 若fd大多数都是就绪的时候, 跟select比起来, 差不多, 因为此时内核需要遍历的就绪列表跟全部fd就差不多了.

一颗红黑树，一张准备就绪fd链表，少量的内核cache，就帮我们解决了大并发下的fd处理问题。

1. 执行epoll_create时，创建了红黑树和就绪list链表(ready list).

2. 执行epoll_ctl时，如果增加fd则检查在红黑树中是否存在，存在立即返回，不存在则添加到红黑树上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪list链表中插入数据.

3. 执行epoll_wait时立刻返回准备就绪链表里的数据即可.


参考: http://blog.csdn.net/kai8wei/article/details/51233494

rcu
=====

linux中的rcu(Read-Copy Update)机制: https://www.ibm.com/developerworks/cn/linux/l-rcu/

关于WRITE_ONCE的解释: https://stackoverflow.com/questions/34988277/write-once-in-linux-kernel-lists, 没怎么看懂

linux vsf
============

linux中的vfs是指一套统一的接口, 然后任何实现了该接口的fs都能被挂载到linux, 然后用户态/内核态都可以使用统一的接口去操作file.

vfs还处理了page cache, inode cache, buffer cache等等. vfs是内核的和物理存储交互的一个软件层(layer), 只定义接口, 具体的操作交给具体fs的实现(ext2,3,4, tmpfs等等)

可以把vfs类比于wsgi去理解.



linux wait_queue
====================

https://stackoverflow.com/questions/19942702/the-difference-between-wait-queue-head-and-wait-queue-in-linux-kernel

http://www.xml.com/ldd/chapter/book/ch05.html, Going to Sleep and Awakening和A Deeper Look at Wait Queues这两节

http://guojing.me/linux-kernel-architecture/posts/wait-queue/


----


python创建epoll
=================

python的epoll是个对象

cpython/Modules/selectmodule.c

.. code-block:: c

    static PyTypeObject pyEpoll_Type = {
        pyepoll_new,                                        /* tp_new */
    };


newPyEpoll_Object
=====================

调用epoll_create或者epoll_create1把epoll赋值到对象中


.. code-block:: c

    static PyObject *
    newPyEpoll_Object(PyTypeObject *type, int sizehint, int flags, SOCKET fd)
    {
        pyEpoll_Object *self;
    
        assert(type != NULL && type->tp_alloc != NULL);
        // 分配个内存
        self = (pyEpoll_Object *) type->tp_alloc(type, 0);
        if (self == NULL)
            return NULL;
    
        // 调用epoll_create1函数, 返回一个和ep有关的fd
        // 把这个fd赋值到self.epfd
        if (fd == -1) {
            Py_BEGIN_ALLOW_THREADS
    #ifdef HAVE_EPOLL_CREATE1
            flags |= EPOLL_CLOEXEC;
            if (flags)
                self->epfd = epoll_create1(flags);
            else
    #endif
            self->epfd = epoll_create(sizehint);
            Py_END_ALLOW_THREADS
        }
        else {
            self->epfd = fd;
        }
        // epoll没成功, 释放内存
        if (self->epfd < 0) {
            Py_DECREF(self);
            PyErr_SetFromErrno(PyExc_OSError);
            return NULL;
        }
        // 下面是一些校验是否成功, 省略
    }


python的register
==================

调用epoll_ctl

.. code-block:: c

    static PyObject *
    pyepoll_internal_ctl(int epfd, int op, PyObject *pfd, unsigned int events)
    {
        struct epoll_event ev;
        int result;
        int fd;
    
        if (epfd < 0)
            return pyepoll_err_closed();
    
        fd = PyObject_AsFileDescriptor(pfd);
        if (fd == -1) {
            return NULL;
        }
    
        // 操作有区别, 但是都是调用epoll_ctl函数
        switch (op) {
        case EPOLL_CTL_ADD:
        case EPOLL_CTL_MOD:
            ev.events = events;
            ev.data.fd = fd;
            Py_BEGIN_ALLOW_THREADS
            result = epoll_ctl(epfd, op, fd, &ev);
            Py_END_ALLOW_THREADS
            break;
        case EPOLL_CTL_DEL:
            /* In kernel versions before 2.6.9, the EPOLL_CTL_DEL
             * operation required a non-NULL pointer in event, even
             * though this argument is ignored. */
            Py_BEGIN_ALLOW_THREADS
            result = epoll_ctl(epfd, op, fd, &ev);
            if (errno == EBADF) {
                /* fd already closed */
                result = 0;
                errno = 0;
            }
            Py_END_ALLOW_THREADS
            break;
        default:
            result = -1;
            errno = EINVAL;
        }
    
        if (result < 0) {
            PyErr_SetFromErrno(PyExc_OSError);
            return NULL;
        }
        Py_RETURN_NONE;
    }

python的poll
=================

这里调用epoll_wait, 然后返回一个受信的fd列表

.. code-block:: c

    static PyObject *
    pyepoll_poll(pyEpoll_Object *self, PyObject *args, PyObject *kwds)
    {
        static char *kwlist[] = {"timeout", "maxevents", NULL};
        PyObject *timeout_obj = NULL;
        int maxevents = -1;
        int nfds, i;
        PyObject *elist = NULL, *etuple = NULL;
        struct epoll_event *evs = NULL;
        _PyTime_t timeout, ms, deadline;
    
        if (self->epfd < 0)
            return pyepoll_err_closed();
    
        if (!PyArg_ParseTupleAndKeywords(args, kwds, "|Oi:poll", kwlist,
                                         &timeout_obj, &maxevents)) {
            return NULL;
        }
    
        // 下面的timeout和deadline都是根据传入的
        // timeout去设置内核函数的timeout参数而已
        if (timeout_obj == NULL || timeout_obj == Py_None) {
            timeout = -1;
            ms = -1;
            deadline = 0;   /* initialize to prevent gcc warning */
        }
        else {
            /* epoll_wait() has a resolution of 1 millisecond, round towards
               infinity to wait at least timeout seconds. */
            if (_PyTime_FromSecondsObject(&timeout, timeout_obj,
                                          _PyTime_ROUND_TIMEOUT) < 0) {
                if (PyErr_ExceptionMatches(PyExc_TypeError)) {
                    PyErr_SetString(PyExc_TypeError,
                                    "timeout must be an integer or None");
                }
                return NULL;
            }
    
            ms = _PyTime_AsMilliseconds(timeout, _PyTime_ROUND_CEILING);
            if (ms < INT_MIN || ms > INT_MAX) {
                PyErr_SetString(PyExc_OverflowError, "timeout is too large");
                return NULL;
            }
    
            deadline = _PyTime_GetMonotonicClock() + timeout;
        }
    
        if (maxevents == -1) {
            maxevents = FD_SETSIZE-1;
        }
        else if (maxevents < 1) {
            PyErr_Format(PyExc_ValueError,
                         "maxevents must be greater than 0, got %d",
                         maxevents);
            return NULL;
        }
    
        // 分个epoll_event的内存
        // 用来存储受信的fd
        evs = PyMem_New(struct epoll_event, maxevents);
        if (evs == NULL) {
            PyErr_NoMemory();
            return NULL;
        }
    
        // 下面的循环才是真正的调用epoll_wait
        do {
            Py_BEGIN_ALLOW_THREADS
            errno = 0;
            nfds = epoll_wait(self->epfd, evs, maxevents, (int)ms);
            Py_END_ALLOW_THREADS
    
            // 如果是被撞断了
            // 说明有fd受信了, 退出循环吧
            if (errno != EINTR)
                break;
    
            /* poll() was interrupted by a signal */
            if (PyErr_CheckSignals())
                goto error;
    
            // 重新计算timeout
            if (timeout >= 0) {
                timeout = deadline - _PyTime_GetMonotonicClock();
                if (timeout < 0) {
                    nfds = 0;
                    break;
                }
                ms = _PyTime_AsMilliseconds(timeout, _PyTime_ROUND_CEILING);
                /* retry epoll_wait() with the recomputed timeout */
            }
        } while(1);
    
        if (nfds < 0) {
            PyErr_SetFromErrno(PyExc_OSError);
            goto error;
        }
    
        // 先分配个list来存储受信的fd
        elist = PyList_New(nfds);
        if (elist == NULL) {
            goto error;
        }
    
        // 组合个list返回给你
        // list包含的是fd和对应的时间, 可以是读写的组合的形式
        for (i = 0; i < nfds; i++) {
            etuple = Py_BuildValue("iI", evs[i].data.fd, evs[i].events);
            if (etuple == NULL) {
                Py_CLEAR(elist);
                goto error;
            }
            PyList_SET_ITEM(elist, i, etuple);
        }
    
        error:
        PyMem_Free(evs);
        return elist;
    }


----


epoll的实现
================

python只是调了epoll的对应函数而已, 具体实现在内核中


epoll_event
=================

这个结构存储了

.. code-block:: c

    struct epoll_event {
    	__u32 events;
    	__u64 data;
    } EPOLL_PACKED;


event_poll
==============

这是epoll结构体

http://elixir.free-electrons.com/linux/v4.15/source/fs/eventpoll.c#L186


.. code-block:: c

    struct eventpoll {
    	/* Protect the access to this structure */
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
    	wait_queue_head_t wq;
    
    	/* Wait queue used by file->poll() */
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
        // 看注释说, ovflist存储的是需要拷贝到用户态的epitem, 是一个单链表
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


epitem
========

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
    
    	/* List header used to link this item to the "struct file" items list */
    	struct list_head fllink;
    
    	/* wakeup_source used when EPOLLWAKEUP is set */
    	struct wakeup_source __rcu *ws;
    
    	/* The structure that describe the interested events and the source fd */
    	struct epoll_event event;
    };

1. 保存红黑树节点的作用是: 查询红黑树的时候, 可以通过已知的红黑树节点的地址通过计算内存其在epitem中的地址偏移量, 反过来得到epitem的地址(参考ep_find)


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
----------------------

event_poll_fops是一套ep定义的file操作接口, 其实就是原生的文件操作接口

file_operations包含的就是vfs的标准接口的集合

.. code-block:: c

    // 定义了read, write等文件操作接口
    // http://elixir.free-electrons.com/linux/v4.15/source/include/linux/fs.h#L1692
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
        // 直接赋值了下面三个函数
    	.release	= ep_eventpoll_release,
    	.poll		= ep_eventpoll_poll,
    	.llseek		= noop_llseek,
    };

所以epoll本身也是一个支持poll的文件, 其poll函数是ep_eventpoll_poll.

anon_inode_getfile
----------------------

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
---------------

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
        // 注意, 这里会把用户态的epoll_event复制到这里, 也就是内核态
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
        // 所以就是检查fd对应的file是否有效, 并且是否支持poll的操作
        error = -EINVAL;
        if (f.file == tf.file || !is_file_epoll(f.file))
        	goto error_tgt_fput;
            
        // 下面是关于exclusive的判断, 没看懂, 省略

    
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
    	if (tep != NULL)
    		mutex_unlock(&tep->mtx);
    	mutex_unlock(&ep->mtx);
    
    error_tgt_fput:
    	if (full_check)
    		mutex_unlock(&epmutex);
    
    	fdput(tf);
    error_fput:
    	fdput(f);
    error_return:
    
    	return error;
    }

1. 检查操作码.

2. 如果不是删除操作, 那么把用户态的 **epoll_event** 拷贝到内核态.

3. 检查操作的fd是否有效, 有效则调用ep_find去查找epoll中红黑树是否包含该fd.

4. 调用插入等操作函数.

fd有效条件包括:

1. 不能是epoll本身, 也就是不能把epoll加入到自己中, 强调自己是因为epoll对应的fd也可以加入到其他epoll中, 因为
   epoll对应的fd也继承了event_poll_fops这些操作.

2. fd对应的file一定实现了event_poll_fops的操作, 最重要的是poll操作.

ep_op_has_event
-----------------

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
-----------

epoll中存放fd的结构是ep_item, 红黑树使得fd的查找最坏也能打到O(logN)


比较的时候需用组织成ffd结构, 然后通过ffd生成一个epitem结构(这里其实就是把ffd设置到epitem中, 当然还包括其他信息), 然后再比较epitem中的ffd.

其实ffd里面就包含两个属性, 一个file, 一个fd

rbp获取epitem
------------------

由于epitem中保存了对应的rbp, 所以可以通过rbp获取对应的epitem:

.. code-block:: c

    epi = rb_entry(rbp, struct epitem, rbn);

    // rb_entry的定义: http://elixir.free-electrons.com/linux/v4.15/source/include/linux/rbtree.h#L66

    #define  rb_entry(ptr, type, member) container_of(ptr, type, member)
   
rb_entry最终调用到的时候container_of, 这个宏的意思是通过计算内存地址的偏移量, 可以通过属性得到整个结构体的地址.

比如epitem是包含了rbn属性的, 所以知道了rbn的地址, 可以计算rbn在整个结构体的偏移量, 得到epitem的地址.

container_of的参考: https://stackoverflow.com/questions/15832301/understanding-container-of-macro-in-the-linux-kernel


比较过程
------------

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
    
        // 下面各种双链表的初始化没看懂
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
        // 这里调用的ep_item_poll就是调用目标文件的poll操作
        // 如果受信, 则返回受信的事件, revents可以是POLLIN等单个事件, 或者组合事件
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
        // 下面是获取自旋锁, 然后把epi的fllink加入到目标文件的poll的监听链表中
        // 可以这么理解, 把epi加入到file的poll的回调链表中
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
    
        // 下面就是这个判断插入之后, 是否就受信了
        // 如果是, 直接把fd添加到epoll的就绪链表中
    	/* If the file is already "ready" we drop it inside the ready list */
    	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
    		list_add_tail(&epi->rdllink, &ep->rdllist);
    		ep_pm_stay_awake(epi);
    
    		/* Notify waiting tasks that events are available */
    		if (waitqueue_active(&ep->wq))
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
    
    error_remove_epi:
    	spin_lock(&tfile->f_lock);
    	list_del_rcu(&epi->fllink);
    	spin_unlock(&tfile->f_lock);
    
    	rb_erase_cached(&epi->rbn, &ep->rbr);
    
    error_unregister:
    	ep_unregister_pollwait(ep, epi);
    
    	/*
    	 * We need to do this because an event could have been arrived on some
    	 * allocated wait queue. Note that we don't care about the ep->ovflist
    	 * list, since that is used/cleaned only inside a section bound by "mtx".
    	 * And ep_insert() is called with "mtx" held.
    	 */
    	spin_lock_irqsave(&ep->lock, flags);
    	if (ep_is_linked(&epi->rdllink))
    		list_del_init(&epi->rdllink);
    	spin_unlock_irqrestore(&ep->lock, flags);
    
    	wakeup_source_unregister(ep_wakeup_source(epi));
    
    error_create_wakeup_source:
    	kmem_cache_free(epi_cache, epi);
    
    	return error;
    }
