Thread
=============

python代码实现
=================

_active_limbo_lock
----------------------

threading中的_active_limbo_lock是一个模块级别的变量, 也就是说每一次对 **进程中任一thread状态** 的操作都要加锁, 同一时间只有操作一个thread的active/limbo状态

1 _active字典是当前进程所有活跃的thread的字典, _limbo字典是当前进入所有准备启动(已初始化完成)的字典

2. _active和_limbo字典是互斥的:

   1.1.2 当thread初始化但是没有run的时候, 把thread对象放在_limbo字典中, 如果thread开始run了,

   1.1.3 那么把thread对象从_limbo字典移除, 然后加入到_active字典中

self._tstate_lock
------------------

这是每一个线程的 **状态锁**, 这个锁是一旦thread状态需要变更了, 那么这个锁就会被设置上, 然后thread终止, 底层C代码会

删除对应thread的state, 然后解释器会释放掉这个锁, 说明这个线程已经终止了. 在C代码没有释放这个锁之前, 对thread的stop操作是禁止的

可以隔段时间(timeout, 或者select/epoll)去检查self.alive是否为True(类似于gunicorn进程模式中子进程判断self.parent, 协程序判断self.alive这种),

但是如果线程卡在一个计算密集型的任务, 就没法去检查self.alive了

所以一般建议线程也不要去执行计算密集型的操作, 否则join不掉也杀不掉

.. code-block:: python

    # threading.Thread
    class Thread:

        def _set_tstate_lock(self):
            """
            Set a lock object which will be released by the interpreter when
            the underlying thread state (see pystate.h) gets deleted.
            """
            # 这里是设置_tstate_lock的地方, 然后注释上说这个锁会在底层的thread状态被删除的时候被解释器释放掉
            self._tstate_lock = _set_sentinel()
            self._tstate_lock.acquire()

如果self._tstate_lock没有被释放, 那么证明线程还是处于被调度状态, 调用self._stop会报错, 而调用self.join同样也是必须去等待这个锁被释放

self._stop和self.join参考下面

**python2的threading.Thread有一个__stop方法, 但是在python3中被去掉了**


daemon
----------

1. 只能等所有的非daemon线程都完成之后, 整个python进程才会退出, 比如非daemon子线程还在sleep, 那么就算主线程已经return了, 那么整个进程还是会等那个子线程完成才退出的

2. daemon thread是一个后台线程, 一旦主线程退出, 那么解释器会等待所有的non-daemon线程退出然后退出, 其他daemon线程就不管了:

   *A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left. The initial value is inherited from the creating thread. The flag can be set through the daemon property or the daemon constructor argument.*

   2.1 如果子线程中又创建一个子线程, 如果那个线程是daemon/non-daemon, 如何退出?

       不管谁创建了子线程, 都是属于同一个进程的, 那么退出机制还是一样的, 主线程会等待所有的non-daemon线程退出, 不管daemon线程

3. 如果daemon由于主线程退出被杀死的话, 资源可能不会被正确释放:

   *Daemon threads are abruptly stopped at shutdown. Their resources (such as open files, database transactions, etc.) may not be released properly. If you want your threads to stop gracefully, make them non-daemonic and use a suitable signalling mechanism such as an Event.*


start
----------

开启一个pthread

.. code-block:: python

    class Thread:
    
        def start(self):
            """Start the thread's activity.
    
            It must be called at most once per thread object. It arranges for the
            object's run() method to be invoked in a separate thread of control.
    
            This method will raise a RuntimeError if called more than once on the
            same thread object.
    
            """
            '''
	    注释上就说明了start是开启一个os的线程, 然后当thread被调度的时候, 会执行
	    self.run里面的代码, 但其实这里做了一些封装, 其实是调用self._bootstrap, 里面才会调用self.run
            '''
            if not self._initialized:
                raise RuntimeError("thread.__init__() not called")
    
            if self._started.is_set():
                # 不能start多次
                raise RuntimeError("threads can only be started once")
            # 这里把thread先放到_limbo字典, 这里_active_limbo_lock这个锁
            with _active_limbo_lock:
                _limbo[self] = self
            try:
                # 这里_start_new_thread是c代码, 然后执行的方法是self._bootstrap 
                _start_new_thread(self._bootstrap, ())
            except Exception:
                with _active_limbo_lock:
                    del _limbo[self]
                raise
            # 这里会阻塞直到self._started这个信号被受信
            # self._started会在self._bootstrap_inner被调用才被受信
            # 也就是说只有self._bootstrap被调用, _bootstrap会直接调用_bootstrap_inner, 才算被调度了
            self._started.wait()

_bootstrap
-------------

这个方法是thread被os调度的时候执行的方法


注释里面说明了try, raise的意义, 也就是说如果thread是_daemonic的, 那么有可能在某一个错误的时间点(at an unfortunate moment),

daemon线程被调度的时候发现解释器环境已经被回收清除了(finds the world around it destroyed), 然后会产生一些

随机的异常(and raises some random exception), 但是daemon线程是一些"无所谓的"线程, 是不太在意的, 那么此时这些异常

并不会有任何帮助(the random exception don`t help anyobody), 所以当thread是daemon, 然后发生异常的时候, 这里会直接return

如果thread不是daemon的, 那么会raise


**这里注意的是, return的条件是thread是daemon, 并且解释器已经被清除, 解释器被清除的判断条件是_sys is None, 所以如果一个daemon thread
在解释器没有被清除的时候, 发生了异常, 依然会raise**

.. code-block:: python

    def _bootstrap(self):
        # Wrapper around the real bootstrap code that ignores
        # exceptions during interpreter cleanup.  Those typically
        # happen when a daemon thread wakes up at an unfortunate
        # moment, finds the world around it destroyed, and raises some
        # random exception *** while trying to report the exception in
        # _bootstrap_inner() below ***.  Those random exceptions
        # don't help anybody, and they confuse users, so we suppress
        # them.  We suppress them only when it appears that the world
        # indeed has already been destroyed, so that exceptions in
        # _bootstrap_inner() during normal business hours are properly
        # reported.  Also, we only suppress them for daemonic threads;
        # if a non-daemonic encounters this, something else is wrong.
        try:
            self._bootstrap_inner()
        except:
            # _sys is None表示解释器已经被清除回收了
            if self._daemonic and _sys is None:
                return
            raise


_bootstrap_inner
-------------------

真正执行逻辑的地方

设置线程锁, 把线程放到活跃线程的dict中, 运行self.run


.. code-block:: python

    def _bootstrap_inner(self):
        try:
            self._set_ident()
            # 这里设置一个tstate_lock, 一些层的c代码
            self._set_tstate_lock()
            # 设置当前线程已经开始了, 这里在self.start
            # 里面的调用的self._started.wait就会返回了
            self._started.set()
            # 这里操作_active或者_limbo字典, 获取一下_active_limbo_lock
            with _active_limbo_lock:
                _active[self._ident] = self
                del _limbo[self]

            if _trace_hook:
                _sys.settrace(_trace_hook)
            if _profile_hook:
                _sys.setprofile(_profile_hook)

            try:
                # 这里调用了self.run, self.run会调用self.target, 也就是用户的逻辑
                self.run()
            except SystemExit:
                pass
            except:
                # If sys.stderr is no more (most likely from interpreter
                # shutdown) use self._stderr.  Otherwise still use sys (as in
                # _sys) in case sys.stderr was redefined since the creation of
                # self.
                # 处理重定向逻辑和打印异常
                if _sys and _sys.stderr is not None:
                    print("Exception in thread %s:\n%s" %
                          (self.name, _format_exc()), file=_sys.stderr)
                elif self._stderr is not None:
                    # Do the best job possible w/o a huge amt. of code to
                    # approximate a traceback (code ideas from
                    # Lib/traceback.py)
                    exc_type, exc_value, exc_tb = self._exc_info()
                    try:
                        print((
                            "Exception in thread " + self.name +
                            " (most likely raised during interpreter shutdown):"), file=self._stderr)
                        print((
                            "Traceback (most recent call last):"), file=self._stderr)
                        while exc_tb:
                            print((
                                '  File "%s", line %s, in %s' %
                                (exc_tb.tb_frame.f_code.co_filename,
                                    exc_tb.tb_lineno,
                                    exc_tb.tb_frame.f_code.co_name)), file=self._stderr)
                            exc_tb = exc_tb.tb_next
                        print(("%s: %s" % (exc_type, exc_value)), file=self._stderr)
                    # Make sure that exc_tb gets deleted since it is a memory
                    # hog; deleting everything else is just for thoroughness
                    finally:
                        del exc_type, exc_value, exc_tb
            finally:
                # Prevent a race in
                # test_threading.test_no_refcycle_through_target when
                # the exception keeps the target alive past when we
                # assert that it's dead.
                #XXX self._exc_clear()
                pass
        finally:
            with _active_limbo_lock:
                try:
                    # We don't call self._delete() because it also
                    # grabs _active_limbo_lock.
                    # 这里把当前thread从活动thread字典中删除
                    del _active[get_ident()]
                except:
                    pass


_stop
--------

**py2的threading.Thread.__stop方法已经在py3中被删除了**

_stop不应该被主动调用的, 因为没有办法或者主动stop一个线程是不好的

调用_stop的时候如果发现self._tstate_lock没有被释放, 那么会报错, 说明不允许主动stop一个thread


注释上意思就是self._stop调用的时候, self._tstate_lock一定是被释放掉的状态


.. code-block:: python

    def _stop(self):
        # After calling ._stop(), .is_alive() returns False and .join() returns
        # immediately.  ._tstate_lock must be released before calling ._stop().
        #
        # Normal case:  C code at the end of the thread's life
        # (release_sentinel in _threadmodule.c) releases ._tstate_lock, and
        # that's detected by our ._wait_for_tstate_lock(), called by .join()
        # and .is_alive().  Any number of threads _may_ call ._stop()
        # simultaneously (for example, if multiple threads are blocked in
        # .join() calls), and they're not serialized.  That's harmless -
        # they'll just make redundant rebindings of ._is_stopped and
        # ._tstate_lock.  Obscure:  we rebind ._tstate_lock last so that the
        # "assert self._is_stopped" in ._wait_for_tstate_lock() always works
        # (the assert is executed only if ._tstate_lock is None).
        #
        # Special case:  _main_thread releases ._tstate_lock via this
        # module's _shutdown() function.
        lock = self._tstate_lock
        # 这个判断就是判断self._tstate_lock是否被释放了, 或者被重新绑定, 但是没有被上锁
        if lock is not None:
            assert not lock.locked()
        self._is_stopped = True
        self._tstate_lock = None


关于stop子线程嘛: https://stackoverflow.com/questions/323972/is-there-any-way-to-kill-a-thread-in-python

其实这个思路和curio的cancel差不多, 都是往task(thread/coroutine)里面加入excpetion, 当被调度到的时候, 检查下当前

的task是否有异常, 是不是被终止或者被cancel(CancellError), 是的话, 引发然后退出.

`dramatiq <https://github.com/allenling/magne/tree/master/magne/thread_worker/how_rabbitpy_dramatiq_works.rst>`_ 方式也是这样的

这样引发异常的好处是可以让线程可以在异常的时候去clean up.


join
-------

join其实就是等待self._tstate_lock这个锁被释放, 然后调用下_stop, 设置终止状态


.. code-block:: python

    def join(self, timeout=None):
        if not self._initialized:
            raise RuntimeError("Thread.__init__() not called")
        if not self._started.is_set():
            raise RuntimeError("cannot join thread before it is started")
        if self is current_thread():
            raise RuntimeError("cannot join current thread")

        # 等待self._tstate_lock释放
        if timeout is None:
            self._wait_for_tstate_lock()
        else:
            # the behavior of a negative timeout isn't documented, but
            # historically .join(timeout=x) for x<0 has acted as if timeout=0
            self._wait_for_tstate_lock(timeout=max(timeout, 0))

    def _wait_for_tstate_lock(self, block=True, timeout=-1):
        # Issue #18808: wait for the thread state to be gone.
        # At the end of the thread's life, after all knowledge of the thread
        # is removed from C data structures, C code releases our _tstate_lock.
        # This method passes its arguments to _tstate_lock.acquire().
        # If the lock is acquired, the C code is done, and self._stop() is
        # called.  That sets ._is_stopped to True, and ._tstate_lock to None.
        # 注释上就是说thread终止之后, C代码会释放掉self._tstate_lock这个锁
        lock = self._tstate_lock
        if lock is None:  # already determined that the C code is done
            assert self._is_stopped
        # 这里就是等待self._tstate_lock释放了
        elif lock.acquire(block, timeout):
            # 拿到锁之后释放, 彻底结束了
            lock.release()
            # 调用下_stop下设置终止状态
            self._stop()

线程C代码实现
=============================

bootstate结构体
----------------------

bootstate是当前线程和解释器状态, 以及线程要执行的python函数及其参数

线程执行的时候会从这个结构中拿到对应的数据


.. code-block:: c

    // cpython/Modules/_threadmodule.c
    struct bootstate {
        // 解释器状态
        PyInterpreterState *interp;
        // 以下是线程要执行的python函数和参数等等
        PyObject *func;
        PyObject *args;
        PyObject *keyw;
        // 线程状态
        PyThreadState *tstate;
    };



_thread._start_new_thread
-------------------------------

threading.Thread.start调用的是_thread._start_new_thread, 指向函数thread_PyThread_start_new_thread

C代码中pthread运行的函数是_bootstrap!!这要注意下

cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    thread_PyThread_start_new_thread(PyObject *self, PyObject *fargs)
    {
        PyObject *func, *args, *keyw = NULL;
        struct bootstate *boot;
        unsigned long ident;
        // 省略了很多check代码
        // 分配一个boot内存
        boot = PyMem_NEW(struct bootstate, 1);
        if (boot == NULL)
            return PyErr_NoMemory();
        //保存当前解释器的状态
        boot->interp = PyThreadState_GET()->interp;
        // boot结构保存了要执行的python函数和参数
        boot->func = func;
        boot->args = args;
        boot->keyw = keyw;
        //预分配一个线程状态结构的内存
        boot->tstate = _PyThreadState_Prealloc(boot->interp);
        if (boot->tstate == NULL) {
            PyMem_DEL(boot);
            return PyErr_NoMemory();
        }
        // 增加func等的引用计数
        Py_INCREF(func);
        Py_INCREF(args);
        Py_XINCREF(keyw);
 
        // py初始化线程, 这里如果是主线程, 那么是去创建gil
        PyEval_InitThreads(); /* Start the interpreter's thread-awareness */
   
        // 去创建一个线程, 并返回ident
        // 这里面注意的是, 执行的函数是t_bootstrap这个, 这个函数基本是执行python函数, 然后做thread清理工作的!!
        ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);
 
        if (ident == PYTHREAD_INVALID_THREAD_ID) {
            PyErr_SetString(ThreadError, "can't start new thread");
            Py_DECREF(func);
            Py_DECREF(args);
            Py_XDECREF(keyw);
            PyThreadState_Clear(boot->tstate);
            PyMem_DEL(boot);
            return NULL;
        }
        return PyLong_FromUnsignedLong(ident);
    }


PyEval_InitThreads
-----------------------

cpython/Python/ceval.c

功能主要是: 如果当前是主线程的初始化, 那么创建gil, 并且获取gil

如果不是主线程的初始化, 那么会在调用t_bootstrap的时候去获取gil

.. code-block:: c

    void
    PyEval_InitThreads(void)
    {
        // 如果已经创建了gil, 退出
        if (gil_created())
            return;
        // 创建gil, 并且抢gil
        create_gil();
        take_gil(PyThreadState_GET());
        _PyRuntime.ceval.pending.main_thread = PyThread_get_thread_ident();
        if (!_PyRuntime.ceval.pending.lock)
            _PyRuntime.ceval.pending.lock = PyThread_allocate_lock();
    }



PyThread_start_new_thread
------------------------------

cpython/Python/thread_pthread.h

最后创建线程是调用pthread_create

.. code-block:: c

    unsigned long
    PyThread_start_new_thread(void (*func)(void *), void *arg)
    {
        pthread_t th;
        int status;
        //下面省略了很多#if
        if (!initialized)
            // 是否初始化了, 这个initialized是否是全局的, 只是主线程start的时候会初始化一次
            // 还是说每次新创建线程的时候都会初始化一次?
            PyThread_init_thread();
        // 又省略了一些#if
 
        // 这里是真正创建线程的地方, 调用系统调用pthread_create
        // 带有很多编译if
        // th就是thred的ident了
        // func传递到第3个参数, 表示thread要执行什么函数
        // func是t_bootstrap
        status = pthread_create(&th,
        #if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
                             &attrs,
        #else
                             (pthread_attr_t*)NULL,
        #endif
                             (void* (*)(void *))func,
                             (void *)arg
                             );
    }

PyThread_init_thread 
-----------------------

这个函数最终调用到PyThread__init_thread, 不过一般都直接退出的

cpython/Python/thread_pthread.h

.. code-block:: c

    static void
    PyThread__init_thread(void)
    {
    #if defined(_AIX) && defined(__GNUC__)
        extern void pthread_init(void);
        // 这里貌似是只有aix平台下并且使用gnu的c编译器才会调用pthread_init
        // 反正没找到pthread_init这个函数的详细信息
        pthread_init();
    #endif
    }

pthread执行的t_bootstrap
---------------------------

t_bootstrap中会运行boot结构中的记录的python函数, 在python代码中就是self._bootstrap

t_bootstrap最后return之前还会释放掉self._tstate_lock这个锁, 这个锁是记录在thread state中的on_delete_data中

并且是调用thread state中的on_delete(on_delete_data), 也就是release_sentinel(on_delete_data)来释放锁

cpython/Modules/_threadmodule.c

.. code-block:: python

    static void
    t_bootstrap(void *boot_raw)
    {
        struct bootstate *boot = (struct bootstate *) boot_raw;
        // 声明下thread的tstate结构
        PyThreadState *tstate;
        PyObject *res;
    
        tstate = boot->tstate;
        tstate->thread_id = PyThread_get_thread_ident();
        // 未清楚目的
        _PyThreadState_Init(tstate);
        // 这里是拿gil, 最终调用的是take_gil这个函数
        PyEval_AcquireThread(tstate);

        // 当前进程(解释器)的总线程数+1
        tstate->interp->num_threads++;

        // 这里调用python的函数, 包含了gil的竞争!!!
        res = PyObject_Call(boot->func, boot->args, boot->keyw);
        if (res == NULL) {
            if (PyErr_ExceptionMatches(PyExc_SystemExit))
                PyErr_Clear();
            else {
                PyObject *file;
                PyObject *exc, *value, *tb;
                PySys_WriteStderr(
                    "Unhandled exception in thread started by ");
                PyErr_Fetch(&exc, &value, &tb);
                file = _PySys_GetObjectId(&PyId_stderr);
                if (file != NULL && file != Py_None)
                    PyFile_WriteObject(boot->func, file, 0);
                else
                    PyObject_Print(boot->func, stderr, 0);
                PySys_WriteStderr("\n");
                PyErr_Restore(exc, value, tb);
                PyErr_PrintEx(0);
            }
        }
        else
            Py_DECREF(res);
        // 清理python函数和参数
        Py_DECREF(boot->func);
        Py_DECREF(boot->args);
        Py_XDECREF(boot->keyw);
        // 释放掉boot结构
        PyMem_DEL(boot_raw);
        // 减少进程的总线程数
        tstate->interp->num_threads--;
        // 清理掉tstate这个thread state
        PyThreadState_Clear(tstate);
        // 删除当前进程的state
        PyThreadState_DeleteCurrent();
        // 终止线程
        PyThread_exit_thread();
    }

1. _PyThreadState_Init这个作用没有完全清楚, 跟_PyRuntime.gilstate.autoTSSkey结构有关

2. PyEval_AcquireThread这个函数最终是调用take_gil去获取gil

PyObject_Call
----------------

这个调用_PyEval_EvalFrameDefault去是执行opcode的过程, 参考: python_gil.rst

PyThreadState_Clear
-------------------------------

这个函数是清理tstate: cpython/Python/pystate.c

是把tstate的结构给释放掉了, 情况tstate结构, 没什么好看的

PyThreadState_DeleteCurrent
------------------------------

主要是删除tstate中的锁, 这样py代码的join会返回

.. code-block:: c

    void
    PyThreadState_DeleteCurrent()
    {
        PyThreadState *tstate = GET_TSTATE();
        if (tstate == NULL)
            Py_FatalError(
                "PyThreadState_DeleteCurrent: no current tstate");

        // 这个函数将会去删除tstate中的锁
        tstate_delete_common(tstate);

        // 下面都是判断gil的状态
        if (_PyRuntime.gilstate.autoInterpreterState &&
            PyThread_tss_get(&_PyRuntime.gilstate.autoTSSkey) == tstate)
        {
            PyThread_tss_set(&_PyRuntime.gilstate.autoTSSkey, NULL);
        }
        SET_TSTATE(NULL);
        // 释放gil
        PyEval_ReleaseLock();
    }

tstate_delete_common
----------------------

删除tstate的锁

.. code-block:: c

    tstate_delete_common(PyThreadState *tstate)
    {
        PyInterpreterState *interp;
        if (tstate == NULL)
            Py_FatalError("PyThreadState_Delete: NULL tstate");
        interp = tstate->interp;
        if (interp == NULL)
            Py_FatalError("PyThreadState_Delete: NULL interp");
        HEAD_LOCK();
        if (tstate->prev)
            tstate->prev->next = tstate->next;
        else
            interp->tstate_head = tstate->next;
        if (tstate->next)
            tstate->next->prev = tstate->prev;
        HEAD_UNLOCK();
        // 这里是调用on_delete是删除on_delete_data
        if (tstate->on_delete != NULL) {
            tstate->on_delete(tstate->on_delete_data);
        }
        PyMem_RawFree(tstate);
    }

HEAD_LOCK定义是:

.. code-block:: c

    #define HEAD_LOCK() PyThread_acquire_lock(_PyRuntime.interpreters.mutex, WAIT_LOCK)

所以是去获取_PyRuntime.interpreters.mutex这个锁, 显然这个锁是一个排它锁, 只能有一个线程获取它, 作用是什么现在还没搞懂


self._tstate_lock
--------------------

在python代码中, 调用self._bootstrap_inner的时候会首先调用self._set_tstate_lock来设置self._tstate_lock这个锁

.. code-block:: python

    def _set_tstate_lock(self):
        # 调用_thread._set_sentinel
        self._tstate_lock = _set_sentinel()
        self._tstate_lock.acquire()

_thread._set_sentinel为thread__set_sentinel, 设置线程state被清除时候的clean up数据

cpython/Modules/_threadmodule.c

.. code-block:: c

    static PyObject *
    thread__set_sentinel(PyObject *self)
    {
        PyObject *wr;
        // 获取线程状态
        PyThreadState *tstate = PyThreadState_Get();
        // 声明锁对象
        lockobject *lock;
    
        if (tstate->on_delete_data != NULL) {
            /* We must support the re-creation of the lock from a
               fork()ed child. */
            # 子进程的tstate继承于父进程的tstate, 所以这里需要重新初始化一下
            assert(tstate->on_delete == &release_sentinel);
            wr = (PyObject *) tstate->on_delete_data;
            tstate->on_delete = NULL;
            tstate->on_delete_data = NULL;
            Py_DECREF(wr);
        }
        // 初始化锁对象
        lock = newlockobject();
        if (lock == NULL)
            return NULL;
        /* The lock is owned by whoever called _set_sentinel(), but the weakref
           hangs to the thread state. */
        // wr指向锁对象
        wr = PyWeakref_NewRef((PyObject *) lock, NULL);
        if (wr == NULL) {
            Py_DECREF(lock);
            return NULL;
        }
        // 这里当thread终止的时候会调用release_sentinel这个方法删除wr这个锁
        tstate->on_delete_data = (void *) wr;
        tstate->on_delete = &release_sentinel;
        // 这里最后返回的是一个锁, 只是之前已经把锁保存到thread状态里面了
        return (PyObject *) lock;
    }

release_sentinel
~~~~~~~~~~~~~~~~~~

cpython/Modules/_threadmodule.c

t_bootstrap函数最后调用tstate_delete_common函数做一些清除操作的时候被调用

release_sentinel这个函数是在tstate->on_delete_data被删除的时候调用的

.. code-block:: c

    static void
    release_sentinel(void *wr)
    {
        /* Tricky: this function is called when the current thread state
           is being deleted.  Therefore, only simple C code can safely
           execute here. */
        // 注释上说的就是当前线程的state被删除的时候调用release_sentinel
        // 传入wr就是tstate->on_delete_data保存的锁
        PyObject *obj = PyWeakref_GET_OBJECT(wr);
        lockobject *lock;
        if (obj != Py_None) {
            assert(Py_TYPE(obj) == &Locktype);
            // C的知识忘了, 这里应该是把obj强制转换成lockobject这个类型
            lock = (lockobject *) obj;
            if (lock->locked) {
                // 释放掉tstate->on_delete_data这个锁
                PyThread_release_lock(lock->lock_lock);
                lock->locked = 0;
            }
        }
        /* Deallocating a weakref with a NULL callback only calls
           PyObject_GC_Del(), which can't call any Python code. */
        Py_DECREF(wr);
    }

PyThread_release_lock这个函数的用户是释放掉锁, 详细实现参考: python_thread_sync_primitive.rst

这个函数调用完成之后, python代码里面的join所调用的self._wait_for_tstate_lock会返回, 此时thread的生命周期已经完全结束

