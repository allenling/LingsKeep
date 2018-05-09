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

