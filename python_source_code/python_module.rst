#############
import/module
#############

python中的import和module


字节码
===========

先从字节码流程看看, 首先是 *import module*

.. code-block:: python

    In [9]: dis.dis("import os")
      1           0 LOAD_CONST               0 (0)
                  2 LOAD_CONST               1 (None)
                  4 IMPORT_NAME              0 (os)
                  6 STORE_NAME               0 (os)
                  8 LOAD_CONST               1 (None)
                 10 RETURN_VALUE


在看看IMPORT_NAME的代码

.. code-block:: c

        TARGET(IMPORT_NAME) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *fromlist = POP();
            PyObject *level = TOP();
            PyObject *res;
            res = import_name(f, name, fromlist, level);
            Py_DECREF(level);
            Py_DECREF(fromlist);
            SET_TOP(res);
            if (res == NULL)
                goto error;
            DISPATCH();
        }

然后是from import语句: *from module import func* 和 *from moudule.another import func*

.. code-block:: python

    In [15]: dis.dis("from mytest import func")
      1           0 LOAD_CONST               0 (0)
                  2 LOAD_CONST               1 (('func',))
                  4 IMPORT_NAME              0 (mytest)
                  6 IMPORT_FROM              1 (func)
                  8 STORE_NAME               1 (func)
                 10 POP_TOP
                 12 LOAD_CONST               2 (None)
                 14 RETURN_VALUE
    
    In [16]: dis.dis("from mytest.other import func")
      1           0 LOAD_CONST               0 (0)
                  2 LOAD_CONST               1 (('func',))
                  4 IMPORT_NAME              0 (mytest.other)
                  6 IMPORT_FROM              1 (func)
                  8 STORE_NAME               1 (func)
                 10 POP_TOP
                 12 LOAD_CONST               2 (None)
                 14 RETURN_VALUE


所以关键就是import_name这个函数

import_name
================


.. code-block:: c

    static PyObject *
    import_name(PyFrameObject *f, PyObject *name, PyObject *fromlist, PyObject *level)
    {
        _Py_IDENTIFIER(__import__);
        PyObject *import_func, *res;
        PyObject* stack[5];
    
        import_func = _PyDict_GetItemId(f->f_builtins, &PyId___import__);
        if (import_func == NULL) {
            PyErr_SetString(PyExc_ImportError, "__import__ not found");
            return NULL;
        }
    
        /* Fast path for not overloaded __import__. */
        if (import_func == PyThreadState_GET()->interp->import_func) {
            int ilevel = _PyLong_AsInt(level);
            if (ilevel == -1 && PyErr_Occurred()) {
                return NULL;
            }
            res = PyImport_ImportModuleLevelObject(
                            name,
                            f->f_globals,
                            f->f_locals == NULL ? Py_None : f->f_locals,
                            fromlist,
                            ilevel);
            return res;
        }
    
        Py_INCREF(import_func);
    
        stack[0] = name;
        stack[1] = f->f_globals;
        stack[2] = f->f_locals == NULL ? Py_None : f->f_locals;
        stack[3] = fromlist;
        stack[4] = level;
        res = _PyObject_FastCall(import_func, stack, 5);
        Py_DECREF(import_func);
        return res;
    }



module对象
=============

python中, module也是一个对象

cpython/Objects/moduleobject.c

.. code-block:: c

    typedef struct {
        PyObject_HEAD
        PyObject *md_dict;
        struct PyModuleDef *md_def;
        void *md_state;
        PyObject *md_weaklist;
        PyObject *md_name;  /* for logging purposes after md_dict is cleared */
    } PyModuleObject;


**其中, md_dict就是module可访问的对象了**

new一个module
=================

cpython/Objects/moduleobject.c

.. code-block:: c

    PyObject *
    PyModule_NewObject(PyObject *name)
    {
        PyModuleObject *m;
        m = PyObject_GC_New(PyModuleObject, &PyModule_Type);
        if (m == NULL)
            return NULL;
        m->md_def = NULL;
        m->md_state = NULL;
        m->md_weaklist = NULL;
        m->md_name = NULL;
        m->md_dict = PyDict_New();
        if (module_init_dict(m, m->md_dict, name, NULL) != 0)
            goto fail;
        PyObject_GC_Track(m);
        return (PyObject *)m;
    
     fail:
        Py_DECREF(m);
        return NULL;
    }


导入module
===============



