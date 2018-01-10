gil实现代码
=============

文章可以参考: python_gil_dabeaz.rst

take_gil
===========

等待一个timeout, 然后强制其他线程释放

cpython/Python/ceval_gil.h


.. code-block:: c 

    static void take_gil(PyThreadState *tstate)
    {
        int err;
        if (tstate == NULL)
            Py_FatalError("take_gil: NULL tstate");
    
        err = errno;
        MUTEX_LOCK(_PyRuntime.ceval.gil.mutex);
    
        if (!_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked))
            goto _ready;
    
        while (_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked)) {
 
            // 如果没有拿到gil
            int timed_out = 0;
            unsigned long saved_switchnum;
            // 这个保存起来switchnum, 没有变化就是gil没有被drop

            saved_switchnum = _PyRuntime.ceval.gil.switch_number;

            // 这个就是等待INTERVAL(5ms)
            COND_TIMED_WAIT(_PyRuntime.ceval.gil.cond, _PyRuntime.ceval.gil.mutex,
                            INTERVAL, timed_out);
            /* If we timed out and no switch occurred in the meantime, it is time
               to ask the GIL-holding thread to drop it. */
            // 注释上说的就是timeout了但是gil没有switch, 判断条件就是switch_number没有变化
            // 然后发出一个drop gil请求, 然后继续while去获取
            if (timed_out &&
                _Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked) &&
                _PyRuntime.ceval.gil.switch_number == saved_switchnum) {
                SET_GIL_DROP_REQUEST();
            }
        }
    }

_PyEval_EvalFrameDefault
==========================

执行python代码的时候一般都是调用PyObject_Call这个函数, 然后经过漫长的跟踪, 总于, 找到真正执行python frame的函数, 

那么就是位于python/Python/ceval.c中, 函数名为_PyEval_EvalFrameDefault

这里省略了很多东西, 后面再补充

主要来看执行python代码, 严格来说是opcode, 的时候如何drop/take gil


注意的是, 这里for执行python的frame的时候, 首先是查看是否有drop gil request发出, 所以

调用_PyEval_EvalFrameDefault这个函数的时候必须是先拿到gil的

比如在_thread.t_bootstrap就是先拿gil, 然后调用PyObject_Call这个函数, 然后调用到这里

.. code-block:: c

    PyObject* _Py_HOT_FUNCTION
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 省略了一堆东西, 后面在补充吧
        // 这个for就是一步一步的去执行opcode了
        for (;;) {
            // 又省略了一堆东西, 后面再补充
            // 这个if是判断是否收到一个gil_drop_request, 有就去drop
            if (_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil_drop_request))
            {
                if (PyThreadState_Swap(NULL) != tstate)
                    Py_FatalError("ceval: tstate mix-up");
                // 首先drop_gil
                drop_gil(tstate);
                // 然后take_gil
                take_gil(tstate);
            }
        }
    }



