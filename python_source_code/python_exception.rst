##########
python异常
##########


存储异常
================

一个获取不存在属性的例子:

.. code-block:: python

    class A:
        pass
    
    a = A()
    
    print(a.a)


显然, 比如发生异常, 看看如何引发和存储异常, 通过字节码知道, *getattr* 是LOAD_ATTR


.. code-block:: c

    TARGET(LOAD_ATTR) {
        PyObject *name = GETITEM(names, oparg);
        PyObject *owner = TOP();
        PyObject *res = PyObject_GetAttr(owner, name);
        Py_DECREF(owner);
        SET_TOP(res);
        if (res == NULL)
            goto error;
        DISPATCH();
    }


PyObject_GetAttr中, 会优先调用tp_getattro, 然后是tp_getattr, 一般自定义的对象都是tp_getattro, 并且tp_getattro = PyObject_GenericGetAttr

所以, 调用关系就是 PyObject_GetAttr -> PyObject_GenericGetAttr -> _PyObject_GenericGetAttrWithDict

在_PyObject_GenericGetAttrWithDict中, 判断异常:

.. code-block:: c

    PyObject *
    _PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict)
    {
    
        // 显然, 前面的代码执行成功, 会跑到done
        // 否则就顺序执行到PyErr_Format
        PyErr_Format(PyExc_AttributeError,
                     "'%.50s' object has no attribute '%U'",
                     tp->tp_name, name);
      done:
        Py_XDECREF(descr);
        Py_DECREF(name);
        return res;
    
    }


所以, PyExc_Format就赋值存储异常了, 调用顺序是:

.. code-block:: python
   
    '''
    
    PyExc_Format -> PyErr_FormatV -> PyErr_Clear                        -> PyErr_Restore(NULL, NULL, NULL)
                                  
                                  -> PyErr_SetObject(exception, string) -> PyErr_Restore(exception, value, tb);
                                                                            
    
    
    
    '''

所以就是, PyErr_Restore是存储异常的地方, 而PyErr_FormatV先是调用PyErr_Clear去通过传入NULL去清除tstate上的老的异常信息

然后调用PyErr_SetObject, 通过传入具体的值给PyErr_Restore去设置当前tstate的异常信息

cpython/Python/error.c

.. code-block:: c

    void
    PyErr_Restore(PyObject *type, PyObject *value, PyObject *traceback)
    {
        PyThreadState *tstate = PyThreadState_GET();
        PyObject *oldtype, *oldvalue, *oldtraceback;
    
        if (traceback != NULL && !PyTraceBack_Check(traceback)) {
            /* XXX Should never happen -- fatal error instead? */
            /* Well, it could be None. */
            Py_DECREF(traceback);
            traceback = NULL;
        }
    
        /* Save these in locals to safeguard against recursive
           invocation through Py_XDECREF */
        oldtype = tstate->curexc_type;
        oldvalue = tstate->curexc_value;
        oldtraceback = tstate->curexc_traceback;
    
        tstate->curexc_type = type;
        tstate->curexc_value = value;
        tstate->curexc_traceback = traceback;
    
        Py_XDECREF(oldtype);
        Py_XDECREF(oldvalue);
        Py_XDECREF(oldtraceback);
    }



判断异常
==============

比如在shell模式下,　去判断当前tstate是否保存了异常

.. code-block:: c

    int
    PyRun_InteractiveLoopFlags(FILE *fp, const char *filename_str, PyCompilerFlags *flags)
    {
        do {
            ret = PyRun_InteractiveOneObjectEx(fp, filename, flags);
            // 判断是否有异常, ret==-1, 并且PyErr_Occurred返回不是NULL
            if (ret == -1 && PyErr_Occurred()) {

                // 打印异常信息    
            }
    
        }
    
    }
    
    // 异常存储在tstate->curexc_type中
    PyObject *
    PyErr_Occurred(void)
    {
        PyThreadState *tstate = _PyThreadState_UncheckedGet();
        return tstate == NULL ? NULL : tstate->curexc_type;
    }


异步异常
=============

在python中, 线程无法主动停止, 但是python提供了一个插入异步异常的api

异步异常会在每次要执行opcode的时候, 去判断


当然, 如果线程是卡在计算中, 那么自然也无法被引发异常, 所以是"异步异常"嘛

在解释器执行函数_PyEval_EvalFrameDefault中 

.. code-block:: c

    for (;;) {
    
        // 显然, 解释器的执行需要被中断
        // 为什么呢,　在if块中判断
        if (_Py_atomic_load_relaxed(&eval_breaker)) {
        
            // 签名还会判断是不是因为信号, 是不是因为有drop_gil_request
            
            // 是因为有异步异常
            if (tstate->async_exc != NULL) {
            
                // 拿出异步异常
                // 注意的是, 拿出异步异常之后, tstate上的异步异常就被清除了
                PyObject *exc = tstate->async_exc;
                tstate->async_exc = NULL;
                UNSIGNAL_ASYNC_EXC();
                PyErr_SetNone(exc);
                Py_DECREF(exc);
                goto error;
            
            }
        
        }
        
        // 执行opcode
    
    }






