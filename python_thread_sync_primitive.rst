#####################################
这里是threading这个库的同步原语的实现
#####################################

一个锁就支持了所有原语的实现!

Lock
======

threading.Lock在C代码是用一个计数为1的semaphore来实现的, 其中代码里面的PyThread_type_lock是指向任意值的指针

.. code-block:: c

    // cpython/include/pythread.h
    typedef void *PyThread_type_lock;

acquire lock, 也就是调用sem_wait(还有其他sem函数)的时候可能:

1. 成功(PY_LOCK_ACQUIRED), 返回值为0

2. 失败(PY_LOCK_FAILURE), 超时也会返回失败的

3. 被中断了(PY_LOCK_INTR), 被中断表示其他线程调用了sem_post, semaphore的计数为1了, 这个时候可以去抢了

sem_post则是把semaphore的计数加1, 然后发送一个中断, 好让sem_wait(等等)函数捕获到


创建新的lock object
----------------------

cpython/Modules/_threadmodule.c

.. code-block:: c

    // lockobject的结构体
    typedef struct {
        PyObject_HEAD
        PyThread_type_lock lock_lock;
        PyObject *in_weakreflist;
        char locked; /* for sanity checking */
    } lockobject;

    static PyObject *
    thread_PyThread_allocate_lock(PyObject *self)
    {
        return (PyObject *) newlockobject();
    }

    // 创建lock的函数
    static lockobject *
    newlockobject(void)
    {
        lockobject *self;
        self = PyObject_New(lockobject, &Locktype);
        if (self == NULL)
            return NULL;
        // 分配锁的内存
        self->lock_lock = PyThread_allocate_lock();
        // 是否被锁住
        self->locked = 0;
        self->in_weakreflist = NULL;
        if (self->lock_lock == NULL) {
            Py_DECREF(self);
            PyErr_SetString(ThreadError, "can't allocate lock");
            return NULL;
        }
        return self;
    }

PyThread_allocate_lock
------------------------

linux下就是pthread的allocate_lock

cpython/Python/thread_pthread.h

基本上就是分配个lock结构出来

.. code-block:: c

    PyThread_type_lock
    PyThread_allocate_lock(void)
    {
        // sem_t是一个semaphore结构体
        sem_t *lock;
        int status, error = 0;
    
        dprintf(("PyThread_allocate_lock called\n"));
        // 或许可以初始化一下线程
        if (!initialized)
            PyThread_init_thread();
        // 这个分配下锁结构体的内存 
        lock = (sem_t *)PyMem_RawMalloc(sizeof(sem_t));
    
        if (lock) {
            // 这里sme_init就是初始化一个semaphore
            // 第二个参数为0表示是在线程之前共享, 1表示是进程间共享
            // 最后一个参数就是semaphore的初始化个数, 这里为1
            status = sem_init(lock,0,1);
            CHECK_STATUS("sem_init");
    
            if (error) {
                PyMem_RawFree((void *)lock);
                lock = NULL;
            }
        }
    
        dprintf(("PyThread_allocate_lock() -> %p\n", lock));
        // 返回个lock对象指针吧?c语言忘得差不多了, 不太记得了
        // 好像void表示的是无类型指针, 所以可以返回一个指向任意类型的指针, 好像是这样
        return (PyThread_type_lock)lock;
    }



acquire 
---------------

cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    lock_PyThread_acquire_lock(lockobject *self, PyObject *args, PyObject *kwds)
    {
        _PyTime_t timeout;
        PyLockStatus r;
    
        // 反正就是判断下参数
        if (lock_acquire_parse_args(args, kwds, &timeout) < 0)
            return NULL;
        // acquire_timed才是真正的acquire地方
        r = acquire_timed(self->lock_lock, timeout);
        if (r == PY_LOCK_INTR) {
            return NULL;
        }
    
        if (r == PY_LOCK_ACQUIRED)
            self->locked = 1;
        return PyBool_FromLong(r == PY_LOCK_ACQUIRED);
    }

acquire_timed
----------------

cpython/Modules/_threadmodule.c


.. code-block:: c

    static PyLockStatus
    acquire_timed(PyThread_type_lock lock, _PyTime_t timeout)
    {
        // 如果timeout大于0, 那么计算一下结束时间
        if (timeout > 0)
            endtime = _PyTime_GetMonotonicClock() + timeout;
    
        do {
            // 这里先直接去那锁, 万一就拿到了呢?
            r = PyThread_acquire_lock_timed(lock, 0, 0);
            if (r == PY_LOCK_FAILURE && microseconds != 0) {
                // 这里的话, 直接获取锁失败了~~~
                // 保存线程状态, 然后调用PyThread_acquire_lock_timed
                Py_BEGIN_ALLOW_THREADS
                r = PyThread_acquire_lock_timed(lock, microseconds, 1);
                Py_END_ALLOW_THREADS
    	    }
        }
    }

这里的Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS这两个宏呢得一起使用, 把中间的代码给包起来的, 目的是中间代码执行之前

保存当前进程的状态, 释放gil, 执行后加载线程状态, 获取gil, 参考 `这里 <https://docs.python.org/3/c-api/init.html#c.Py_BEGIN_ALLOW_THREADS`_

Py_BEGIN_ALLOW_THREADS是定义了这样一个代码块:

.. code-block:: c

    { PyThreadState *_save; _save = PyEval_SaveThread();

注意是没有}花括号的, 需要Py_END_ALLOW_THREADS来关闭花括号, 然后PyEval_SaveThread是保存状态然后释放gil的:

cpython/Python/ceval.c

.. code-block:: c

    PyEval_SaveThread(void)
    {
        PyThreadState *tstate = PyThreadState_Swap(NULL);
        if (tstate == NULL)
            Py_FatalError("PyEval_SaveThread: NULL tstate");
        if (gil_created())
            // 这里会释放掉gil
            drop_gil(tstate);
        return tstate;
    }



PyThread_acquire_lock_timed
-----------------------------

这个函数就是一直while去获取锁的了

cpython/Python/thread_pthread.h

.. code-block:: c

    PyLockStatus
    PyThread_acquire_lock_timed(PyThread_type_lock lock, PY_TIMEOUT_T microseconds,
                                int intr_flag)
    {
        while (1) {
            // 根据不同的timeout调用不同的系统调用
            if (microseconds > 0) {
                status = fix_status(sem_timedwait(thelock, &ts));
            }
            else if (microseconds == 0) {
                status = fix_status(sem_trywait(thelock));
            }
            else {
                status = fix_status(sem_wait(thelock));
            }
    
           if (intr_flag || status != EINTR) {
               // 这里表示status返回了, 但是不是EINTR, 也就是说acquire有结果了, 退出
               // 如果status是EINTR, 则表示sem_post发出了中断, semaphore计数加1了, 接下来需要去抢锁了
               break;
           }
    
           // 这里是收到中断, 然后继续抢锁之前, 如果有超时, 就要计算超时时间的deadline
           if (microseconds > 0) {
              // 这里就省略了吧
           }
 
           /* Retry if interrupted by a signal, unless the caller wants to be
              notified.  */
           // 这里如果status==EINTR, 也就是收到中断, 直接继续
           if (intr_flag || status != EINTR) {
               break;
           }
  
           // 下面是计算超时的, 省略了
           if (microseconds > 0) {
           }
       }
    
       /* Don't check the status if we're stopping because of an interrupt.  */
       // 这里的注释说, while循环被打破了, 如果是因为一个中断被打破的, why?或许是如果是被中断打断, 必然是没拿到锁吧
       // 如果不是中断, 那么检查下：
       // 是否是sem_timedwait超时了还是拿到了锁
       // 是否是调用sem_trywait, sem_trywait是立即返回的, 拿到了锁
       // 是否是sem_waits拿到了锁
       if (!(intr_flag && status == EINTR)) {
           if (microseconds > 0) {
               if (status != ETIMEDOUT)
                   CHECK_STATUS("sem_timedwait");
           }
           else if (microseconds == 0) {
               if (status != EAGAIN)
                   CHECK_STATUS("sem_trywait");
           }
           else {
               CHECK_STATUS("sem_wait");
           }
       }
    
    
       // 设置success, sem_这些调用返回0的时候表示拿到了semaphore
       if (status == 0) {
           success = PY_LOCK_ACQUIRED;
       } else if (intr_flag && status == EINTR) {
           success = PY_LOCK_INTR;
       } else {
           success = PY_LOCK_FAILURE;
       }
    }

所以这里的意思就是, 比如sem_wait这个系统调用, 是一直减少指向锁的信号量的计数的, 在之前的代码中lockobject被初始化为线程间共享, 并且计数为1

这里如果有很多thread同时acquire的话, 如果lock是被锁住的, 那么对应的semaphore的计数就是0, 然后release的时候, 调用的系统调用是sem_post, 计数器加1, 同时发出中断

然后此时sem_wait将会得到中断, 也就是error.EINTR, 然后会去竞争semaphore, 如果竞争不到, 继续, 直到有返回值.

release
-----------

release比较简单, 调用sem_post释放semaphore就好了

cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    lock_PyThread_release_lock(lockobject *self)
    {
        /* Sanity check: the lock must be locked */
        if (!self->locked) {
            PyErr_SetString(ThreadError, "release unlocked lock");
            return NULL;
        }
        // 这里真正释放锁的地方 
        PyThread_release_lock(self->lock_lock);
        self->locked = 0;
        Py_RETURN_NONE;
    }

PyThread_release_lock: cpython/Python/thread_pthread.h

.. code-block:: c

    PyThread_release_lock(PyThread_type_lock lock)
    {
        sem_t *thelock = (sem_t *)lock;
        int status, error = 0;
    
        (void) error; /* silence unused-but-set-variable warning */
        dprintf(("PyThread_release_lock(%p) called\n", lock));
        // 这里调用下sem_post这个系统调用 
        status = sem_post(thelock);
        CHECK_STATUS("sem_post");
    }


RLock
========


可重入锁对象, 一个已经acquire了rlock对象的线程, 可以再次acquire, 此时rlock的个数加1

threading中, 如果_thread中未定义RLock, 那么RLock对象是一个python代码实现的rlock, 如果定义了_thread.RLock, 那么
threading.RLock返回的是一个C定义的rlcok.

其实流程上来说, python实现的RLock和C实现的RLock差不多, 都是初始化一个owner和count, 判断owner以及增减count

.. code-block:: python

    # 查看是否定义有_thread.RLock
    try:
        _CRLock = _thread.RLock
    except AttributeError:
        _CRLock = None

    def RLock(*args, **kwargs):
        # 如果CRLock未定义, 那么使用一个python实现的RLock
        if _CRLock is None:
            return _PyRLock(*args, **kwargs)
        return _CRLock(*args, **kwargs)


python实现的RLock
-------------------

python实现的RLock是threading.RLock类

.. code-block:: python

    class _RLock:
        def __init__(self):
            # 一个lock, 一个owner一个count
            self._block = _allocate_lock()
            self._owner = None
            self._count = 0

        def acquire(self, blocking=True, timeout=-1):
            me = get_ident()
            # acquire的时候判断是否是自己
            if self._owner == me:
                # 是自己的话count加1
                self._count += 1
                return 1
            # 不是自己的话去获取lock, 然后设置owner
            rc = self._block.acquire(blocking, timeout)
            if rc:
                self._owner = me
                self._count = 1
            return rc

        def release(self):
            # 自己未获取rlock, 不能释放
            if self._owner != get_ident():
                raise RuntimeError("cannot release un-acquired lock")
            # 释放的时候count减1
            self._count = count = self._count - 1
            if not count:
                self._owner = None
                self._block.release()


C实现的RLock
---------------

流程差不多, 只是是C代码实现的而已

cpython/Modules/_threadmodule.c

.. code-block:: c

    // rlock的结构体, 同时有owner和count属性
    typedef struct {
        PyObject_HEAD
        PyThread_type_lock rlock_lock;
        unsigned long rlock_owner;
        unsigned long rlock_count;
        PyObject *in_weakreflist;
    } rlockobject;

acquire
------------


cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    rlock_acquire(rlockobject *self, PyObject *args, PyObject *kwds)
    {
        _PyTime_t timeout;
        unsigned long tid;
        PyLockStatus r = PY_LOCK_ACQUIRED;
    
        if (lock_acquire_parse_args(args, kwds, &timeout) < 0)
            return NULL;
    
        tid = PyThread_get_thread_ident();
        // 查看owner是不是自己
        if (self->rlock_count > 0 && tid == self->rlock_owner) {
            # owner是自己, 则count加1
            unsigned long count = self->rlock_count + 1;
            # 这个时候如果count越界的话, 那么count就小于self->rlock_count了
            if (count <= self->rlock_count) {
                PyErr_SetString(PyExc_OverflowError,
                                "Internal lock count overflowed");
                return NULL;
            }
            self->rlock_count = count;
            Py_RETURN_TRUE;
        }
        // owner不是自己的话就去acquire, 然后还有个timeout
        // 拿锁还是调用acquire_timed
        r = acquire_timed(self->rlock_lock, timeout);
        // acquire成功, 设置下owner和count
        if (r == PY_LOCK_ACQUIRED) {
            assert(self->rlock_count == 0);
            self->rlock_owner = tid;
            self->rlock_count = 1;
        }
        else if (r == PY_LOCK_INTR) {
            return NULL;
        }
    
        return PyBool_FromLong(r == PY_LOCK_ACQUIRED);
    }

release
---------

cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    rlock_release(rlockobject *self)
    {
        unsigned long tid = PyThread_get_thread_ident();
    
        // 如果自己没有拿锁, raise
        if (self->rlock_count == 0 || self->rlock_owner != tid) {
            PyErr_SetString(PyExc_RuntimeError,
                            "cannot release un-acquired lock");
            return NULL;
        }
        // 减少一下计数, 然后设置owner=0
        if (--self->rlock_count == 0) {
            self->rlock_owner = 0;
            PyThread_release_lock(self->rlock_lock);
        }
        Py_RETURN_NONE;
    }


Condition
===========

控制访问, 基本上是存储子锁, self.waiters, 然后释放self.waiters里面的锁来通知其他线程的

notify就是释放一个(semaphore), notify_all就是就是释放全部(event)

这里需要借助其他同步变量来理解

Event
======

**Event也是用Condition来实现**, set的是就是notifiy_all来唤醒所有的线程


初始化
--------

event是由带一个互斥锁的condition, 和一个flag来实现的


.. code-block:: python

    # threading.Event
    class Event:
        def __init__(self):
            # 这里有带了一个互斥锁的Condition
            self._cond = Condition(Lock())
            self._flag = False

所以每次set/wait的时候, 必然要调用Condition的notify_all/wait, 那么此时必然会锁住condition._lock

那么两个线程一个掉set, 一个调wait的时候, 应该是互斥的, 但是事实不太一样, 一个线程wait的是, 另外一个还是可以set

也就是说Condition看起来又不是互斥的, 但是Condition带的锁确实互斥锁, 怎么理解?

condtion在哪里互斥?
---------------------------

set和wait都会调用Condition.\_\_enter\_\_, 那么会互斥吗?

.. code-block:: python

    # threading.Event.wait
    def wait(self, timeout=None):
        # 你看, 这里会调用with self._cond
        with self._cond:
            # flag如果是True的话, 立马返回
            signaled = self._flag
            if not signaled:
                signaled = self._cond.wait(timeout)
            return signaled

    # threading.Event.set
    def set(self):
        # 这里也会调用with self._cond
        with self._cond:
            self._flag = True
            self._cond.notify_all()

所以看起来, 对于同一个event变量event, t1线程调用event.set, 会阻塞另外一个线程的set(包括wait), 但是事实看起来"没有阻塞".

如果debug进去的话，可以看到当一个线程调用set, 然后阻塞的时候, 另外一个线程调用set, 可以看到, **此时的self._cond的lock却是unlocked的~~说明wait的时候, 必然释放了锁!!!**

wait释放锁互斥锁
--------------------


event.wait会调用condtion.wait, Condition.wait里面就释放了互斥锁了.


下面的Condition._is_owned和Condition._release_save这两个方法只有在Condition._lock不存在这两个方法的时候, 才会调用到,

否则Condition._is_owned和Condition._release_save会调用到Condition._lock._is_owned和Condition._lock._release_save

而互斥锁没有这两个方法, RLock有这两个方法, event中的Condition是带互斥锁的


.. code-block:: python

    # threading.Condition._release_save
    def _release_save(self):
        # -------------这里释放了锁!!!!!!
        self._lock.release()   

    # threading.Condition._is_owned
    def _is_owned(self):
        if self._lock.acquire(0):
            self._lock.release()
            return False
        else:
            return True

    # threading.Condition.wait
    def wait(self, timeout=None):
        # 校验自己是否拿了锁
        if not self._is_owned():
            raise RuntimeError("cannot wait on un-acquired lock")
        # 分配一个子锁
        waiter = _allocate_lock()
        # 拿到这个子锁
        waiter.acquire()
        # 保存这个子锁
        self._waiters.append(waiter)
        # ---------这里就是释放Condition._lock的地方!!!
        saved_state = self._release_save()
        gotit = False
        # 下面的过程就是在子锁上等待重新上锁了
        # 但是要记得最后一定重新拿Condition._lock锁, 否则会影响到外层的with self._cond这个语句的释放
        try:
            if timeout is None:
                waiter.acquire()
                gotit = True
            else:
                if timeout > 0:
                    gotit = waiter.acquire(True, timeout)
                else:
                    gotit = waiter.acquire(False)
            return gotit
        finally:
            # 最后记得重新拿锁, 和外层的with self._cond保持一致
            self._acquire_restore(saved_state)
            if not gotit:
                try:
                    self._waiters.remove(waiter)
                except ValueError:
                    pass

set会一直互斥
---------------

set的调用的话, 知道condtion.notify_all调用完成, 才会释放锁, 然后把flag置为True, 其他wait看到True直接返回, 接着一个个去通知等待的线程(确切的说是等待释放的锁)

这也合理, 不然我已经notify所有的waiter了, 然后你又重新wait, 这样就漏掉了一个没有notify了

所以notify_all会检查自己是否拿了锁, 没拿报错




.. code-block:: python

    # threading.Event.set
    def set(self):
        with self._cond:
            # flag为True, 这样set完毕之后的线程如果再wait的话, 立马返回
            self._flag = True
            # 调用Condition.notify_all()
            self._cond.notify_all()
            # 这里最后执行完毕才解锁

**Condition.notify_all中对锁没有操作, 所以如果Condition._lock锁上了的话, 半途是不会解锁的**

.. code-block:: python

    def notify(self, n=1):
        # 这里校验自己是不是拿了锁, 没拿就报错
        if not self._is_owned():
            raise RuntimeError("cannot notify on un-acquired lock")
        # 所有的waiters
        all_waiters = self._waiters
        waiters_to_notify = _deque(_islice(all_waiters, n))
        if not waiters_to_notify:
            return
        for waiter in waiters_to_notify:
            # 一个个去release, 那么其他线程的acquired就返回了
            waiter.release()
            try:
                all_waiters.remove(waiter)
            except ValueError:
                pass

    def notify_all(self):
        # 调用Condition.notify
        self.notify(len(self._waiters))

Semaphore
===========

**Semaphore也是用Condition来实现**

