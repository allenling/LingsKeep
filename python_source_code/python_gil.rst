gil实现代码
=============

gil图解参考: python_gil_dabeaz.rst


gil流程解释
=============

在源码的注释中有了清楚的说明了, 下面是摘抄和简单翻译:

*The GIL is just a boolean variable (gil_locked) whose access is protected by a mutex (gil_mutex), and whose changes are signalled by a condition

variable (gil_cond). gil_mutex is taken for short periods of time,and therefore mostly uncontended.*

GIL是一个真假值(gil_locked), 其访问是被一个互斥锁(gil_mutex)保护的, 然后当释放gil之后, 会通过一个condtion变量(gil_cond)来通知其他线程可以去抢锁了.

也就是gil_cond是说多个线程抢锁, 就是在gil_cond这个condtion变量上等待, 被系统唤醒(取决于内核唤醒机制)的那一个就拿gil了.


*In the GIL-holding thread, the main loop (PyEval_EvalFrameEx) must be able to release the GIL on demand by another thread. A volatile boolean

variable (gil_drop_request) is used for that purpose, which is checked at every turn of the eval loop. That variable is set after a wait of

`interval` microseconds on `gil_cond` has timed out.*


如果一个线程持有gil, 那么在该线程的主循环(main loop, 也就是在PyEval_EvalFrameEx函数中)中, 在其他线程发起释放gil的请求之后, 需要释放gil. 一个

volatile类型的真假值(gil_drop_request)表示是否有释放gil请求. PyEval_EvalFrameEx每执行一个字节码之后, 都会校验这个真假值, 如果为真, 那么有释放gil

的请求发出. 发起一个gil释放请求(设置gil_drop_request为真)是在等待一段时间(interval变量设置的时间, 默认5毫秒)之后, 如果gil仍然未被释放, 那么才会

去发起该请求. gil_drop_request使用volatile, 表示该变量经常变化, 读取的时候应该实时读取而不是缓存读取.


*A thread wanting to take the GIL will first let pass a given amount of time (`interval` microseconds) before setting gil_drop_request. This

encourages a defined switching period, but doesn't enforce it since opcodes can take an arbitrary time to execute.*

请求gil的线程首先会先等待interval微秒(注意, 这里是微秒), 如果gil仍然给没有被释放, 则设置gil_drop_request去发起释放请求


*When a thread releases the GIL and gil_drop_request is set, that thread ensures that another GIL-awaiting thread gets scheduled. It does so by

waiting on a condition variable (switch_cond) until the value of gil_last_holder is changed to something else than its own thread state pointer,

indicating that another thread was able to take the GIL.*

当线程释放gil释放, 并且gil_drop_request被设置为真, 那么表示这次释放gil是"被迫的"(不过释放gil的操作都是等待其他人请求的, 所以基本上都是"被迫的"),

那么释放的线程会保证拿到gil的线程(等待gil的线可能有多个, 但是只能有一个抢到)已经拿到并开始运行了. 这是使用一个switch_cond的condtion变量实现的.

也就是说, a释放gil, 在switch_cond这个condtion变量上等待, b抢到到gil, 通过switch_cond通知a自己拿到了, 然后a进入等待gil阶段. 这样避免了a释放了gil,

如果a直接进入等待gil的阶段, 有可能a又拿到gil了的情况. 这个情况很正常, 因为释放gil是释放互斥锁(gil_mutx), 然后其他人去抢, 然后a释放了互斥锁, 但是锁

被释放这个信号传递到其他线程需要点时间, 而a是直接又抢锁, 所以a很大可能又抢到锁了, 这会导致一个叫护航效应(convey effect)的问题

gil结构
=========

.. code-block:: c

    /* microseconds (the Python API uses seconds, though) */
    #define DEFAULT_INTERVAL 5000
    static unsigned long gil_interval = DEFAULT_INTERVAL;
    #define INTERVAL (gil_interval >= 1 ? gil_interval : 1)

    #undef FORCE_SWITCHING
    #define FORCE_SWITCHING

    static _Py_atomic_int gil_locked = {-1};

    static unsigned long gil_switch_number = 0;

    static COND_T gil_cond;
    static MUTEX_T gil_mutex;
    
    #ifdef FORCE_SWITCHING
    static COND_T switch_cond;
    static MUTEX_T switch_mutex;
    #endif

    static _Py_atomic_int gil_drop_request = {0};


1. gil_locked               : 就是所谓的gil了, -1表示未初始化.

2. gil_switch_number        : 是发生切换的次数, 初始化为0.

3. gil_mutex                : 保护访问gil_locked的互斥锁
   
4. gil_cond                 : 在这个变量上等待就是抢夺gil

5. switch_mutex, switch_cond: 则是用来保证线程能真正切换的

6. interval                 : 是等待多少时间采取发起gil_drop_request, 默认是5000微秒, 也就是5ms

7. gil_drop_request         : 这个真假值就是说是否有释放gil的请求了, 默认-1, 未初始化

注意的是, gil_mutex和gil_cond是一起使用的, **这个就像python中的condtion变量一样, 只不过锁和condtion被分出来了**

也就是不管是获取gil(take_gil)还是释放gil(drop_gil), 都需要拿到gil_mutex. 如果是获取gil, 那么先拿到gil_mutext, 然后发现gil被其他线程拿着,

那么在gil_cond上等待, 等待的同时释放了gil_mutex, 所以可以同时多个线程等待gil释放.

switch_mutex和switch_cond也是一起使用的


其他辅助结构
=================

.. code-block:: c

    static unsigned long gil_switch_number = 0;

    static _Py_atomic_address gil_last_holder = {0};


1. gil_switch_number: gil总切换次数, 如果switch_number有变化, 那么就是等待gil释放期间, 有释放gil的请求, 避免发送多个drop_gil_request.

2. gil_last_holder  : 最后一个拿到gil的线程.

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

拿锁, 然后如果拿不到, 等个interval看看其他线程会不会主动释放, 然后等待结束了还没有人发送过释放请求, 那自己主动发送一个drop_gil_request, 然后继续等待

等待的时候调用了pthread_cond_timedwait这个系统调用, 根据python中Condition实现的推测, pthread_cond_timedwait这个

系统调用会解锁掉mutex, 是得其他线程也可以在gil的cond上等待, 所以可以支持多个线程一起去拿锁

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
            // 这里会调用到pthread_cond_timedwait系统调用, 释放gil_mutex
            // 让其他线程可以释放gil或者等待gil
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
        // 通知其他线程我已经切换到我了, 你可以跑路了
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
=======================

  *These  functions  atomically  release  mutex and cause the calling thread to block on the condition variable cond*
  
  --- 参考man手册

1. pthread_cond_timedwait这个系统调用的行为则是和Python代码里面的Condition一样, 解锁mutex, 然后等待在waiter锁上

2. pthread_cond_timedwait会被pthread_cond_signal唤醒, 但是 **pthread_cond_signal不能保证只唤醒一个线程(特别是多核情况下)**, 所以

   这里用while和一个_Py_atomic_load_relaxed **原子操作** 保证了多个线程被唤醒的时候, 仍然能保证只要一个线程拿到gil锁(设置gil真假值为真), 并且其他

pthread_cond_signal
========================

  *The pthread_cond_signal() function shall unblock at least one of the threads that are blocked on the specified condition variable cond (if any threads are blocked on cond).*
  
  --- 参考man手册

**最少** 唤醒一个线程, 优先级高的就优先唤醒.

*On a multi-processor, it may be impossible for an implementation of pthread_cond_signal() to avoid the unblocking of more than one thread blocked on a condition variable*

*The effect is that more than one thread can return from its call to pthread_cond_wait() or pthread_cond_timedwait() as a result of one call to pthread_cond_signal(). This effect is called "spurious wakeup".*

根据man手册中的例子, pthread_cond_signal在 **多核环境** 下也有可能唤醒多个线程的, 从而发生虚假唤醒

*An added benefit of allowing spurious wakeups is that applications are forced to code a predicate-testing-loop around the condition wait.

This also makes the application tolerate superfluous condition broadcasts or signals on the same condition variable that may be coded in some other part of the application.

The resulting applications are thus more robust. Therefore, IEEE Std 1003.1-2001 explicitly documents that spurious wakeups may occur.*

虚假唤醒的话需要用一个循环包住cond的wait, 然后校验.

**对比起来, python中的Condition则是fifo通知的**

drop_gil
============

drop_gil的行为就可以take_gil相反了, 推测一下也可以了.

COND_SIGNAL这个宏则是调用pthread_cond_signal这个系统调用来唤醒使用pthread_cond_wait的线程


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
    
    // FORCE_SWITCHING模式记得一定要等待switch_cond的通知!!!
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

drop的顺序
===========

drop_gil的时候, cond通知和mutex的释放的顺序是先发送cond通知, 再释放mutex, 或许也可以先释放mutex, 在发送cond通知:

http://blog.csdn.net/yeyuangen/article/details/37593533


_PyEval_EvalFrameDefault
==========================

1. 这个函数会去执行python的代码, 严格来说是执行opcode, 然后这个函数最终调用到的是_PyEval_EvalFrameDefault.

2. _PyEval_EvalFrameDefault这个函数会一直执行, **每执行一个opcode就检查是否有drop request, 有就调用drop_gil**

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

