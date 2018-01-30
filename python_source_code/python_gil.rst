gil实现代码
=============

gil图解参考: python_gil_dabeaz.rst

gil结构
=========

cpython/Python/ceval_gil.h

gil只是一个真假值, 只是其访问是受到gil_mutex的保护, 已经使用gil_cond来通知其他线程拿锁.

强制切换模式, FORCE_SWITCHING, 的意思是释放gil之后, 必须等待其他某个被调度, 并且拿到锁的线程开始执行， 方法是释放

gile的线程在switch_cond这个竞态上等待. 这样可以保证释放了gil的线程不会马上又拿到, 其他线程一定会执行.

FORCE_SWITCHING模式可以缓解convey effect? 看起来可以


.. code-block:: c
   /*
   - The GIL is just a boolean variable (gil_locked) whose access is protected
     by a mutex (gil_mutex), and whose changes are signalled by a condition
     variable (gil_cond). gil_mutex is taken for short periods of time,
     and therefore mostly uncontended.
     // 省略了点注释
    */
    // 上面的注释基本上详细解释了gil, 我没写完

    // !!!!这个其实就是gil的真实身份了!!!
    /* Whether the GIL is already taken (-1 if uninitialized). This is atomic
       because it can be read without any lock taken in ceval.c. */
    static _Py_atomic_int gil_locked = {-1};

    /* This condition variable allows one or several threads to wait until
       the GIL is released. In addition, the mutex also protects the above
       variables. */
    static COND_T gil_cond;
    static MUTEX_T gil_mutex;

    // FORCE_SWITCHING模式的变量
    #ifdef FORCE_SWITCHING
    /* This condition variable helps the GIL-releasing thread wait for
       a GIL-awaiting thread to be scheduled and take the GIL. */
    static COND_T switch_cond;
    static MUTEX_T switch_mutex;
    #endif

gil_locked就是所谓的gil了, -1表示未初始化.

其中每一组一个互斥锁和一个竞态(Condition), 竞态是保证多个线程可以wait, 而互斥锁是保护gil一次只能被一个线程所操作的作用,

可以看成(就是)threading.Condition结构, 只是Condition中的lock被分出来了

**注意的是: 持有gil的线程最少唤醒一个等待gil的线程.**


其他辅助结构
-----------------

.. code-block:: c

    /* Number of GIL switches since the beginning. */
    static unsigned long gil_switch_number = 0;
    /* Last PyThreadState holding / having held the GIL. This helps us know
       whether anyone else was scheduled after we dropped the GIL. */
    static _Py_atomic_address gil_last_holder = {0};

    /* Request for dropping the GIL */
    static _Py_atomic_int gil_drop_request = {0};

2. gil_switch_number: gil总切换次数, 如果switch_number有变化, 那么就是等待gil释放期间, 有释放gil的请求, 避免发送多个drop_gil_request.

3. gil_last_holder  : 最后一个拿到gil的线程.

4. gil_drop_request : 是否有其他线程发起了drop请求

create_gil
============

cpython/Python/ceval_gil.h


.. code-block:: c

    static void create_gil(void)
    {
        MUTEX_INIT(gil_mutex);
    #ifdef FORCE_SWITCHING
        MUTEX_INIT(switch_mutex);
    #endif
        COND_INIT(gil_cond);
    #ifdef FORCE_SWITCHING
        COND_INIT(switch_cond);
    #endif
        _Py_atomic_store_relaxed(&gil_last_holder, 0);
        _Py_ANNOTATE_RWLOCK_CREATE(&gil_locked);
        _Py_atomic_store_explicit(&gil_locked, 0, _Py_memory_order_release);
    }

创建gil的时候, 就是初始化gil_mutex和gil_cond, 然后设置：

1. 最后一个获取gil的线程为0, 表示还没有人拿到gil

2. 设置gil_locked状态是未锁住状态



take_gil
===========

cpython/Python/ceval_gil.h

拿锁, 然后如果拿不到, 等个timeout看看其他线程会不会主动释放, 然后依然拿不到, 发送一个drop_gil_request, 然后继续

等待的时候调用了pthread_cond_timedwait这个系统调用, 根据python中Condition实现的推测, pthread_cond_timedwait这个

系统调用会解锁掉mutex, 是得其他线程也可以在gil的cond上等待.

这里注意下FORCE_SWITCHING的行为

.. code-block:: c 

    static void take_gil(PyThreadState *tstate)
    {
        int err;
        if (tstate == NULL)
            Py_FatalError("take_gil: NULL tstate");
    
        err = errno;
        // 拿互斥锁
        MUTEX_LOCK(gil_mutex);
    
        // 这一句如果判断gil_locked是假, 也就是gil没有被锁住的话
        // 那么直接去拿锁
        if (!_Py_atomic_load_relaxed(&gil_locked))
            // 拿到锁了, 直接跳到_ready
            goto _ready;
    
        while (_Py_atomic_load_relaxed(&gil_locked)) {
            // 没拿到锁, 那么等个timeout
            int timed_out = 0;
            unsigned long saved_switchnum;
    
            // 这里记录下switch_number, 如果在等待期间改变了, 表示其他线程去发送drop request, 就没有必要发了
            saved_switchnum = gil_switch_number;
            // 在竞态上等待
            // 这里会调用到pthread_cond_timedwait系统调用
            COND_TIMED_WAIT(gil_cond, gil_mutex, INTERVAL, timed_out);
            if (timed_out &&
                _Py_atomic_load_relaxed(&gil_locked) &&
                // 超时了, 并且没有抢到锁, 并且期间没有人发drop request
                gil_switch_number == saved_switchnum) {
                // 自己发个drop, 然后继续吧
                SET_GIL_DROP_REQUEST();
            }
        }
    _ready:
    #ifdef FORCE_SWITCHING
        /* This mutex must be taken before modifying gil_last_holder (see drop_gil()). */
        // 拿到gil之后得拿一下switch_mutex, 等下通知drop的线程
        MUTEX_LOCK(switch_mutex);
    #endif
        /* We now hold the GIL */
        _Py_atomic_store_relaxed(&gil_locked, 1);
        _Py_ANNOTATE_RWLOCK_ACQUIRED(&gil_locked, /*is_write=*/1);
    
        if (tstate != (PyThreadState*)_Py_atomic_load_relaxed(&gil_last_holder)) {
            _Py_atomic_store_relaxed(&gil_last_holder, (uintptr_t)tstate);
            // 增加下总的switch_number
            ++gil_switch_number;
        }
    
    #ifdef FORCE_SWITCHING
        // 通知其他线程我已经切换到我了, 你可以方向的跑路了
        COND_SIGNAL(switch_cond);
        MUTEX_UNLOCK(switch_mutex);
    #endif
        if (_Py_atomic_load_relaxed(&gil_drop_request)) {
            RESET_GIL_DROP_REQUEST();
        }
        if (tstate->async_exc != NULL) {
            _PyEval_SignalAsyncExc();
        }
        // 拿到gil之后, 自然最后要释放gil_mutex 
        MUTEX_UNLOCK(gil_mutex);
        errno = err;
    }

pthread_cond_timedwait
-----------------------

*These  functions  atomically  release  mutex and cause the calling thread to block on the condition variable cond*

---参考man手册

pthread_cond_timedwait这个系统调用的行为则是和Python代码里面的Condition一样, 解锁mutex, 然后等待在waiter锁上

pthread_cond_timedwait会被pthread_cond_signal唤醒, 但是 **pthread_cond_signal不能保证只唤醒一个线程(特别是多核情况下)**, 所以

这里用while和一个_Py_atomic_load_relaxed **原子操作** 保证了多个线程被唤醒的时候, 仍然能保证只要一个线程拿到gil锁(设置gil真假值为真), 并且其他

被唤醒的线程还可以继续wait

drop_gil
============

drop_gil的行为就可以take_gil相反了, 推测一下也可以了.

COND_SIGNAL这个宏则是调用pthread_cond_signal这个系统调用来notify_all


.. code-block:: c

    static void drop_gil(PyThreadState *tstate)
    {
        if (!_Py_atomic_load_relaxed(&gil_locked))
            Py_FatalError("drop_gil: GIL is not locked");
        /* tstate is allowed to be NULL (early interpreter init) */
        if (tstate != NULL) {
            /* Sub-interpreter support: threads might have been switched
               under our feet using PyThreadState_Swap(). Fix the GIL last
               holder variable so that our heuristics work. */
            _Py_atomic_store_relaxed(&gil_last_holder, (uintptr_t)tstate);
        }
    
        // 锁一下mutex
        MUTEX_LOCK(gil_mutex);
        _Py_ANNOTATE_RWLOCK_RELEASED(&gil_locked, /*is_write=*/1);
        _Py_atomic_store_relaxed(&gil_locked, 0);
        // 可以其他线程可以去抢gil了
        COND_SIGNAL(gil_cond);
        // 释放下gil_mutx
        MUTEX_UNLOCK(gil_mutex);
    
    // FORCE_SWITCHING模式记得一定要等待switch_cond的通知
    #ifdef FORCE_SWITCHING
        if (_Py_atomic_load_relaxed(&gil_drop_request) && tstate != NULL) {
            MUTEX_LOCK(switch_mutex);
            /* Not switched yet => wait */
            if ((PyThreadState*)_Py_atomic_load_relaxed(&gil_last_holder) == tstate) {
            RESET_GIL_DROP_REQUEST();
                /* NOTE: if COND_WAIT does not atomically start waiting when
                   releasing the mutex, another thread can run through, take
                   the GIL and drop it again, and reset the condition
                   before we even had a chance to wait for it. */
                // 这里等待另外那个拿到gil的线程的通知!!!!
                COND_WAIT(switch_cond, switch_mutex);
        }
            MUTEX_UNLOCK(switch_mutex);
        }
    #endif
    }

pthread_cond_signal
------------------------

*The pthread_cond_signal() function shall unblock at least one of the threads that are blocked on the specified condition variable cond (if any threads are blocked on cond).*

---参考man手册

**最少** 唤醒一个线程, 优先级高的就优先唤醒.

根据man手册中的例子, pthread_cond_signal在多核环境的也有可能唤醒多个线程的~~~从而发生虚假唤醒:

*On a multi-processor, it may be impossible for an implementation of pthread_cond_signal() to avoid the unblocking of more than one thread blocked on a condition variable*

*The effect is that more than one thread can return from its call to pthread_cond_wait() or pthread_cond_timedwait() as a result of one call to pthread_cond_signal(). This effect is called "spurious wakeup".*

虚假唤醒的话需要用一个循环包住cond的wait, 然后校验:

*An added benefit of allowing spurious wakeups is that applications are forced to code a predicate-testing-loop around the condition wait.

This also makes the application tolerate superfluous condition broadcasts or signals on the same condition variable that may be coded in some other part of the application.

The resulting applications are thus more robust. Therefore, IEEE Std 1003.1-2001 explicitly documents that spurious wakeups may occur.*

**相比threading.Condition.notify则是fifo通知的**

drop的顺序
-----------

drop_gil的时候, cond通知和mutex的释放的顺序是先发送cond通知, 再释放mutex, 或者可以先释放mutex, 在发送cond通知:

http://blog.csdn.net/yeyuangen/article/details/37593533


_PyEval_EvalFrameDefault
==========================

这个函数会去执行python的代码, 严格来说是执行opcode, 然后这个函数最终调用到的是_PyEval_EvalFrameDefault

_PyEval_EvalFrameDefault这个函数会一直执行, 然后回去drop/take gil这样~~~~详情看python_gil.rst

因为PyObject_Call之前就调用了PyEval_AcquireThread来获取到了gil, 那么_PyEval_EvalFrameDefault里面for的第一判断是

查看是否有drop gil request发出, 如果在_PyEval_EvalFrameDefault里面首先又去take的话, 就死锁了呀~~~

所以_PyEval_EvalFrameDefault里面首先是查看是否需要drop

cpython/Python/ceval.c


.. code-block:: c

    PyObject* _Py_HOT_FUNCTION
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 省略了一堆opcode的定义什么的
        // 直接看执行过程
        for (;;) {
            // 还是省略了一堆代码
            // 这就是看有没有drop_gil_request
            if (_Py_atomic_load_relaxed(&gil_drop_request)) {
              /* Give another thread a chance */
              if (PyThreadState_Swap(NULL) != tstate)
                  Py_FatalError("ceval: tstate mix-up");
              // 释放掉gil
              drop_gil(tstate);

              /* Other threads may run now */
              // 然后又立即拿gil
              take_gil(tstate);

              /* Check if we should make a quick exit. */
              if (_Py_Finalizing && _Py_Finalizing != tstate) {
                  drop_gil(tstate);
                  PyThread_exit_thread();
              }

              if (PyThreadState_Swap(tstate) != NULL)
                  Py_FatalError("ceval: orphan tstate");
            } 
            // 这里查看是否有调用c接口把异常给发送进来
            /* Check for asynchronous exceptions. */
            if (tstate->async_exc != NULL) {
                PyObject *exc = tstate->async_exc;
                tstate->async_exc = NULL;
                UNSIGNAL_ASYNC_EXC();
                PyErr_SetNone(exc);
                Py_DECREF(exc);
                goto error;
            }

            // 然后下面是执行opcode的, 太多了
            // 用BUILD_TUPLE来举个例子
            switch (opcode) {
                // TARGET就是case了
                TARGET(BUILD_TUPLE) {
                    PyObject *tup = PyTuple_New(oparg);
                    if (tup == NULL)
                        goto error;
                    while (--oparg >= 0) {
                        PyObject *item = POP();
                        PyTuple_SET_ITEM(tup, oparg, item);
                    }
                    PUSH(tup);
                    DISPATCH();
                }
            }
        }
  
关于原子操作
=============

gil中调用了很多像_Py_atomic_load_relaxed带有atomic的函数, 称为原子操作, 代码在cpython/Include/pyatomic.h.

原子操作是参考自C1x(1x可能是11)的实现, 看不懂. 并且源文件里面说了:

*Beware, the implementations here are deep magic.*

