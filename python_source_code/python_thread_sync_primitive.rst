#####################################
这里是threading这个库的同步原语的实现
#####################################

锁的目的是为了防止线程不安全, 也就是多线程下出现数据错误, 所以多线程操作的时候, 必须拿到锁才能操作

一个例子就是, python中的字节码a +=1分为:

1. 获取a值
   
2. tmp=a+1

3. a = tmp

那么, 假设a=2, 当线程x执行到2的时候, 然后当前线程切换到线程y, 此时y也要执行a+=1, 然后

y获取a, a=2, 然后tmp=a+1, 最后a=tmp, 当y执行完了, a的值变为3, 然后切换到线程x, 那么x继续执行字节码的第二步

tmp = a + 1, 然后更新a = tmp, 此时我们希望a=4, 但是由于对同一个资源操作没有限制, 导致覆盖, 此时a=3

1. 锁的结构和系统定义有关, 如果定义了posix sem支持, 那么锁就是count=1的sem, acquire和release都使用sem_*函数去执行

2. 锁的等待有个注意点就是会调用Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS这组宏, 这组宏的目的是

   切换tstate为NULL, 释放gil, 等到锁acquire返回之后, 在获取gil, 设置上tstate

3. 锁和condtion看起来功能差不多, cond是支持多个线程等待通知的, 然后锁的话, 也可以多个线程一起acquire, 然后只有一个线程

   能获取锁, cond则是支持通知所有人, 返回True表示拿到资源控制权.

   但是有个小细节就是, 锁的timeout是针对锁, 而cond中的timeout是针对等待, 也就是说, 访问cond一定是阻塞的, 没有timeout

   比如x, y, z三个线程去获取锁, timeout都是1s, 那么1s之后, x, y, z就会返回False

   如果是a拿到cond, 然后不释放cond的锁, 那么x, y, z在cond上等待, 设置timeout是1s, 则1s之后, x, y, z不会返回

   因为cond的锁的获取是阻塞无timeout的(acquire函数, 不带timeout), cond的timeout是说, x, y, z三个都进入wait状态了, 1s之后

   x, y, z都返回

4. 当然, Condtion也可以为访问锁加上超时, 但是必须显式的调用cond.acquire, cond.acquire直接调用cond.lock.acquire

Lock
======

Lock在linux中具体实现取决于是否配置了POSIX的semaphore支持, 如果配置了, 则Lock是一个计数为1的semaphore实现, 否则是一个用condtion去模拟semaphore

threading.Lock是一个_thread.allocate_lock函数:

.. code-block:: python

    In [28]: threading.Lock?
    Docstring:
    allocate_lock() -> lock object
    (allocate() is an obsolete synonym)
    
    Create a new lock object. See help(type(threading.Lock())) for
    information about locks.
    Type:      builtin_function_or_method

而在threading.py中, 有Lock = allocate_lock

.. code-block:: python

    # 这里, _allocate_lock函数指向_thread.allocate_lock函数
    _allocate_lock = _thread.allocate_lock
    
    // 所以, 这里的Lock = _thread.allocate_lock
    Lock = _allocate_lock
    
    所以, allocate_lock返回的是一个lock object

而_thread是cpython/Modules/_threadmodules.c, 其中allocate_lock则是thread_PyThread_allocate_lock函数

.. code-block:: c

    static PyMethodDef thread_methods[] = {
        {"allocate_lock",           (PyCFunction)thread_PyThread_allocate_lock,
         METH_NOARGS, allocate_doc},
    };

所以调用 *threading.Lock()* 就是调用thread_PyThread_allocate_lock函数


.. code-block:: c

    static PyObject *
    thread_PyThread_allocate_lock(PyObject *self)
    {
        return (PyObject *) newlockobject();
    }

lockobject对象

.. code-block:: c

    typedef struct {
        PyObject_HEAD
        PyThread_type_lock lock_lock;
        PyObject *in_weakreflist;
        char locked; /* for sanity checking */
    } lockobject;

其中, PyThread_type_lock类型的lock_lock属性就是具体的锁了, 但是PyThread_type_lock的类型是一个任意类型的指针!!!!

.. code-block:: c

    typedef void *PyThread_type_lock;

所以这个lock_lock是什么, 依赖于具体实现了, 在linux下, 自然是pthread的lock了, 继续往下看

newlockobject函数

.. code-block:: c

    static lockobject *
    newlockobject(void)
    {
        lockobject *self;
        self = PyObject_New(lockobject, &Locktype);
        if (self == NULL)
            return NULL;

        // 这里具体分配一个lock
        self->lock_lock = PyThread_allocate_lock();
        self->locked = 0;
        self->in_weakreflist = NULL;
        if (self->lock_lock == NULL) {
            Py_DECREF(self);
            PyErr_SetString(ThreadError, "can't allocate lock");
            return NULL;
        }
        return self;
    }
   

所以lock_lock是依赖于具体的PyThread_allocate_lock的实现, python中, 线程包含了pthread和nt(windows平台)的实现, 那么在linux下, 自然是pthread了.

所以这个PyThread_allocate_lock自然是pthread的实现, 在文件cpython/Python/thread_pthread.h中, 并且取决于宏_POSIX_SEMAPHORES

如果定义了盖宏, 那么Lock则是计数为1的semaphore, 否则使用一个condtion去模拟:


.. code-block:: c

    // _POSIX_SEMAPHORES决定了USE_SEMAPHORES宏

    #if (defined(_POSIX_SEMAPHORES) && !defined(HAVE_BROKEN_POSIX_SEMAPHORES) && \
         defined(HAVE_SEM_TIMEDWAIT))
    #  define USE_SEMAPHORES
    #else
    #  undef USE_SEMAPHORES
    #endif

    // 如果定义了posix semaphore支持
    #ifdef USE_SEMAPHORES
    
    PyThread_type_lock
    PyThread_allocate_lock(void)
    {
        // lock是一个sem_t结构
        sem_t *lock;

        sem_t *lock;
        int status, error = 0;

        dprintf(("PyThread_allocate_lock called\n"));
        if (!initialized)
            PyThread_init_thread();

        lock = (sem_t *)PyMem_RawMalloc(sizeof(sem_t));

        if (lock) {
            // 使用sem_init去初始化计数为1
            status = sem_init(lock,0,1);
            CHECK_STATUS("sem_init");

            if (error) {
                PyMem_RawFree((void *)lock);
                lock = NULL;
            }
        }

        dprintf(("PyThread_allocate_lock() -> %p\n", lock));
        return (PyThread_type_lock)lock;
    
    }

    // 没有内置的POSIX semaphore支持, 则使用一个模拟的
    #else /* USE_SEMAPHORES */

    PyThread_type_lock
    PyThread_allocate_lock(void)
    {
        // 一个pthread_lock结构:
        pthread_lock *lock;
        int status, error = 0;
    
        dprintf(("PyThread_allocate_lock called\n"));
        if (!initialized)
            PyThread_init_thread();
    
        lock = (pthread_lock *) PyMem_RawMalloc(sizeof(pthread_lock));
        if (lock) {
            memset((void *)lock, '\0', sizeof(pthread_lock));
            lock->locked = 0;
    
            // 下面就是初始化condtion的过程了
            // 省略代码
        }
    
        dprintf(("PyThread_allocate_lock() -> %p\n", lock));
        return (PyThread_type_lock) lock;
    }

    // 具体的锁结构
    // 使用condtion来实现
    typedef struct {
        char             locked; /* 0=unlocked, 1=locked */
        /* a <cond, mutex> pair to handle an acquire of a locked lock */
        pthread_cond_t   lock_released;
        pthread_mutex_t  mut;
    } pthread_lock;


sem_t的操作
=============

如果配置了_POSIX_SEMAPHORES, 那么锁的操作都是sem_等等函数

acquire lock, 也就是调用sem_wait(还有其他sem函数)的时候可能:

1. 成功(PY_LOCK_ACQUIRED), 返回值为0

2. 失败(PY_LOCK_FAILURE), 超时也会返回失败的

3. 被中断了(PY_LOCK_INTR), 被中断表示其他线程调用了sem_post, semaphore的计数为1了, 这个时候可以去抢了

sem_post则是把semaphore的计数加1, 然后发送一个中断, 好让sem_wait(等等)函数捕获到


acquire 
===============

cpython/Modules/_threadmodule.c

锁方法的定义中, acquire指向lock_PyThread_acquire_lock

.. code-block:: c

    static PyMethodDef lock_methods[] = {
        {"acquire",      (PyCFunction)lock_PyThread_acquire_lock,
         METH_VARARGS | METH_KEYWORDS, acquire_doc},
    };

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
================

cpython/Modules/_threadmodule.c

先看看PyLockStatus, 这个结构枚举了所有拿锁返回的状态

.. code-block:: c

    typedef enum PyLockStatus {
        PY_LOCK_FAILURE = 0,
        PY_LOCK_ACQUIRED = 1,
        PY_LOCK_INTR
    } PyLockStatus;


所以, 0是拿锁失败, 1是拿锁成功, 和被中断, 被中断是用PY_LOCK_INTR表示

.. code-block:: c

    static PyLockStatus
    acquire_timed(PyThread_type_lock lock, _PyTime_t timeout)
    {
        PyLockStatus r;
        _PyTime_t endtime = 0;
        _PyTime_t microseconds;
    
        // 如果timeout, 计算下绝对时间
        if (timeout > 0)
            endtime = _PyTime_GetMonotonicClock() + timeout;
    
        do {
            microseconds = _PyTime_AsMicroseconds(timeout, _PyTime_ROUND_CEILING);
    
            /* first a simple non-blocking try without releasing the GIL */
            // 先别释放GIL, 直接拿锁, 万一一下子就拿到呢
            r = PyThread_acquire_lock_timed(lock, 0, 0);
            // r!=0, 表示没拿到锁, 则直接调用PyThread_acquire_lock_timed去拿锁
            if (r == PY_LOCK_FAILURE && microseconds != 0) {
                // 下面两个宏是去释放GIL的
                Py_BEGIN_ALLOW_THREADS
                r = PyThread_acquire_lock_timed(lock, microseconds, 1);
                Py_END_ALLOW_THREADS
            }
    
            // 如果是被中断了
            if (r == PY_LOCK_INTR) {
                /* Run signal handlers if we were interrupted.  Propagate
                 * exceptions from signal handlers, such as KeyboardInterrupt, by
                 * passing up PY_LOCK_INTR.  */
                // 如果是被信号中断了, 则返回被中断
                if (Py_MakePendingCalls() < 0) {
                    return PY_LOCK_INTR;
                }
    
                /* If we're using a timeout, recompute the timeout after processing
                 * signals, since those can take time.  */
                if (timeout > 0) {
                    timeout = endtime - _PyTime_GetMonotonicClock();
    
                    /* Check for negative values, since those mean block forever.
                     */
                    if (timeout < 0) {
                        r = PY_LOCK_FAILURE;
                    }
                }
            }
        } while (r == PY_LOCK_INTR);  /* Retry if we were interrupted. */
    
        return r;
    }


1. 先别释放GIL, 直接调用PyThread_acquire_lock_timed去立即拿锁, 其中传入的timeout是0, 也就是不管拿没拿到, 立即返回
   这样是说如果一般情况下是不需要等待就可以拿锁的, 所以可以先试一下

2. 如果1没有拿到锁, 则调用PyThread_acquire_lock_timed, 传入timeout去拿锁, 其中需要调用宏Py_BEGIN_ALLOW_THREADS/Py_END_ALLOW_THREADS去释放GIL

3. 如果2中返回值是被中断状态, 那么先判断是不是被信号打断, 是的话返回中断状态给调用者, 如果不是被信号中断状态, 并且timeout>0, 则需要重新计算timeout

这里的Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS这两个宏得一起使用, 把中间的代码给包起来的, 目的是中间代码执行之前

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
=============================

显然, 取决于_POSIX_SEMAPHORES的宏定义, PyThread_acquire_lock_timed实现是不同的

下面是使用semaphore实现的获取过程, 如果是使用conditon来模拟的, 流程也差不多, 只是需要自己的锁一下condition的mutex, 然后根据情况使用

pthread_cond_timedwait等待释放锁通知

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
===========

release比较简单, 这里只看调用sem_post释放semaphore的流程

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
===================

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
===============

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
============


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
=========

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

notify是FIFO顺序释放一个(semaphore), notify_all就是就是释放全部(event)

这里需要借助其他同步变量来理解, 看下面

Event
======

**Event也是用Condition来实现**, set的是就是notifiy_all来唤醒所有的线程


初始化EVENT
=============

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
===========================

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
====================

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
===============

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
            # FIFO顺序release, 那么其他线程的acquired就返回了
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


queue.Queue
=============

初始化包括存储数据的deque(fifo结构), 以及get, put, not_full, not_empty, all_tasks_done等所需要的Condition.

其中not_full, not_empty, all_tasks_done这三个Condition的锁都是指向一个互斥锁self.mutex, 但是其中会有条件的去wait, wait的时候是释放
self.mutex, 所以可以有多个线程去进行get, put. join操作的wait, 但是只有一个能成功.

就是说可以有2个线程a, b去get, a, b都会wait, 后有2个线程c, d去put, c, d都会去wait, 但是同一时间a, b, c, d只有一个可以成功.

**意味着: 获取三个Condition中的任意(只能一个)一个, 也隐式的拿到了其他两个Condition!! 因为三个Condition的lock都是同一个self.mutex!!**

**但是由于是互相调用notify, 所以notify的时候可以通知到不同目的(get/put/join)的线程**

1. a线程去put, 获取了self.not_full这个Cond, 由于not_full这个Cond的lock是self.mutex，所以b线程要get的时候, 要获取not_empty, 由于
   not_empty的lock也是self.mutex, 所以b会被阻塞住

2. a发现deque没有满, 则直接append, 然后释放not_full, 也就是释放self.mutex, 退出, b此时拿到了not_empty, 发现deque不是空, 则直接从deque中拿到数据

3. 假设现在deque是满的, 那么当a要去put的时候, 拿到not_full, 发现deque是满的, 那么在not_full这个Con上等待, 然后释放self.mutex.
   注意, 在not_fully上的Cond上等待, 意味着a是加入到not_full上的waiters上, 释放not_full的时候, b线程的get也拿到锁, 是因为
   not_empty的lock也是self.mutex, 释放not_full就是释放了self.mutext, 而b线程等待的not_empty的lock也是self.mutex, 所以b才能拿到锁
   
4. b拿到not_empty, 然后发现deque不是空, 直接拿一个数据, 然后调用not_full.notify去通知在not_full上等待的线程, 可以put了

5. 由于a是在not_full上等待的, 所以b在get之后调用的not_full.notify就是通知到a, a返回, 说明可以put了.

6. 在deque是空的情况, 就是b在not_empty上等待, 然后a则调用not_empty.notify去通知b可以get了.

.. code-block:: python

    class Queue:
        def __init__(self, maxsize=0):
            self.maxsize = maxsize
            self._init(maxsize)
    
            # 下面是各种Condition
            self.mutex = threading.Lock()
    
            self.not_empty = threading.Condition(self.mutex)
    
            self.not_full = threading.Condition(self.mutex)
    
            self.all_tasks_done = threading.Condition(self.mutex)
            self.unfinished_tasks = 0

        def _init(self, maxsize):
            # 初始化一个fifo结构
            self.queue = deque()

put
======

获取not_full这个Condition, 并且操作完成之前是不会释放掉Condition的, 所以如果没有满, 那么直接_put然后退出解锁

如果满了, 调用Condition.wait去释放锁, 让其他线程有机会去get, 使得queue达到未满的状态, 或者其他线程也一起put进行等待.

最后调用_put去添加数据之后, 调用not_empty.notify去通过可以去get了.

为什么能调用not_empty.notify呢? 因为not_full和not_empty这两个Condition用的是同一个lock对象, 所以获取了一个就相当于

获取了另外一个了.

所以put会调用not_empty.notify, 通知可以get

get调用not_full.notify通知可以put

.. code-block:: python

    def put(self, item, block=True, timeout=None):
        # 拿到not_full的Condition
        with self.not_full:
            if self.maxsize > 0:
                if not block:
                    # 如果是non-block方式, 直接raise异常
                    if self._qsize() >= self.maxsize:
                        raise Full
                elif timeout is None:
                    # ********** 如果是block模式, 并且没有timeout, 则直接去在调用not_full(Condition).wait
                    # *********  这样就释放了锁, 允许其他人去get/put
                    while self._qsize() >= self.maxsize:
                        self.not_full.wait()
                elif timeout < 0:
                    raise ValueError("'timeout' must be a non-negative number")
                else:
                    # block模式且有timeout, 则调用wait(timeout)
                    endtime = time() + timeout
                    while self._qsize() >= self.maxsize:
                        remaining = endtime - time()
                        if remaining <= 0.0:
                            raise Full
                        self.not_full.wait(remaining)
            # 这里如果不限制大小的话, 直接调用_put, 然后退出
            # 不限制大小的话每次都通知别人not_empty, 让别人能get
            self._put(item)
            self.unfinished_tasks += 1
            # ****** 为什么能直接调用not_empty这个Condition.notify呢? 这里并没有去获取not_empty这个Condition
            # ***** 答案就是not_empty和not_full公用一个lock, 所以可以notify
            self.not_empty.notify()

get
=======

和put差不多, 只不过把not_full缓存了not_empty!!


.. code-block:: python

    def get(self, block=True, timeout=None):
        # 获取not_empty
        with self.not_empty:
            if not block:
                if not self._qsize():
                    raise Empty
            elif timeout is None:
                # 这里调用wait释放一下
                while not self._qsize():
                    self.not_empty.wait()
            elif timeout < 0:
                raise ValueError("'timeout' must be a non-negative number")
            else:
                # 无非是wait加个timeout咯
                endtime = time() + timeout
                while not self._qsize():
                    remaining = endtime - time()
                    if remaining <= 0.0:
                        raise Empty
                    self.not_empty.wait(remaining)
            item = self._get()
            # notify只会notify监听not_full的线程!!!
            self.not_full.notify()
            return item

join/task_done
-----------------

差不多的了!!!!

