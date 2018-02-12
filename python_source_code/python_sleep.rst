time.sleep实现
================

python中的time.sleep是使用select等待一个超时来实现的



time_sleep
=============

time的调用是cpython/Modules/timemodule.c中的time_sleep


.. code-block:: c

    static PyObject *
    time_sleep(PyObject *self, PyObject *obj)
    {
        _PyTime_t secs;
        if (_PyTime_FromSecondsObject(&secs, obj, _PyTime_ROUND_TIMEOUT))
            return NULL;
        if (secs < 0) {
            PyErr_SetString(PyExc_ValueError,
                            "sleep length must be non-negative");
            return NULL;
        }
        if (pysleep(secs) != 0)
            return NULL;
        Py_INCREF(Py_None);
        return Py_None;
    }

pysleep
=========

针对不同平台实现不同的sleep, 这里只看非windows平台


.. code-block:: c

    static int
    pysleep(_PyTime_t secs)
    {
        _PyTime_t deadline, monotonic;
    #ifndef MS_WINDOWS
        struct timeval timeout;
        int err = 0;
    #else
    // 这里是windows平台的变量定义
    #endif
    
        deadline = _PyTime_GetMonotonicClock() + secs;
    
        do {
    #ifndef MS_WINDOWS
            if (_PyTime_AsTimeval(secs, &timeout, _PyTime_ROUND_CEILING) < 0)
                return -1;
    
            // 下面两个包裹着select的宏是先释放gil然后获取gil
            Py_BEGIN_ALLOW_THREADS
            // select系统调用
            err = select(0, (fd_set *)0, (fd_set *)0, (fd_set *)0, &timeout);
            Py_END_ALLOW_THREADS
    
            if (err == 0)
                break;
    
            if (errno != EINTR) {
                PyErr_SetFromErrno(PyExc_OSError);
                return -1;
            }
    #else
    // 里面是windows平台的处理
    #endif
    
            /* sleep was interrupted by SIGINT */
            if (PyErr_CheckSignals())
                return -1;
    
            monotonic = _PyTime_GetMonotonicClock();
            secs = deadline - monotonic;
            if (secs < 0)
                break;
            /* retry with the recomputed delay */
        } while (1);
    
        return 0;
    }





