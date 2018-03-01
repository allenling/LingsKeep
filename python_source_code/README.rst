python源码实现
===============

主要是3.6版本


python接口分层
=================

简单分析python中layer的设计:


.. code-block:: python

    '''
    
        +--------------------------------------------------------+
        |                                                        |
        | 编译层                                                 |
        | 这一层主要是把pyton语法转成字节码, 然后调用对应的接口  |
        | 代码中cpython/Python/ceval.c                           |
        |                                                        |
        +--------------------------------------------------------+
        |                                                        |
        | 中间接口                                               |
        | 这一层是提供编译层和具体实现之间的一个中间层           |
        |                                                        |
        +--------------------------------------------------------+
        |                                                        |
        | 对象层, 包括对象的定义和方法的定义, 和gc, 内存管理交互 |
        |                                                        |
        +--------------------------------------------------------+
        |                            |                           |
        | gc操作                     | 内存分配和管理            |
        |                            |                           |
        +----------------------------+---------------------------+
    
    '''


编译过程
==============

python中执行语句之前, 会把语法转成codeobject(这里先跳过语法解析什么的), 然后根据codeobject中的字节码去执行程序.


1. 创建codeobject


比如 **x[1] = 'a'** 这个代码, 执行之前, 会编译生成一个codeobject

.. code-block:: c

    PyCodeObject *
    PyCode_New(int argcount, int kwonlyargcount,
               int nlocals, int stacksize, int flags,
               PyObject *code, PyObject *consts, PyObject *names,
               PyObject *varnames, PyObject *freevars, PyObject *cellvars,
               PyObject *filename, PyObject *name, int firstlineno,
               PyObject *lnotab)
    {
    
        // 省略代码
    
    }


2. 运行codeobject
 
然后运行codeobjetc中的字节码(下面代码是在shell模式下):

.. code-block:: c

    // cpython/Python/pythonrun.c
    static PyObject *
    run_mod(mod_ty mod, PyObject *filename, PyObject *globals, PyObject *locals,
                PyCompilerFlags *flags, PyArena *arena)
    {
    
        // 省略代码
        
        // 这一句就是调用上面的PyCode_New去生成返回codeobject
        co = PyAST_CompileObject(mod, filename, flags, -1, arena);
        
        // 执行codeobject
        v = PyEval_EvalCode((PyObject*)co, globals, locals);
        
        // 省略代码
    
    }

3. 创建frame

执行的时候, 根据当前线程的状态和codeobject, 创建需要执行的frame, 然后执行frame


.. code-block:: c

    // 这个函数是被上面的PyEval_EvalCode调用
    static PyObject *
    _PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
               PyObject **args, Py_ssize_t argcount,
               PyObject **kwnames, PyObject **kwargs,
               Py_ssize_t kwcount, int kwstep,
               PyObject **defs, Py_ssize_t defcount,
               PyObject *kwdefs, PyObject *closure,
               PyObject *name, PyObject *qualname)
    {
    
    // 省略代码
    
        /* Create the frame */
        // 线程状态
        tstate = PyThreadState_GET();
        assert(tstate != NULL);
        // 执行的frame
        f = PyFrame_New(tstate, co, globals, locals);

        // 省略代码

        // 这里执行frame
        retval = PyEval_EvalFrameEx(f,0);
    
        // 省略代码
    
    }


4. 执行frame

执行frame是使用当前解释器去执行


.. code-block:: c


    // cpython/Python/ceval.c
    PyObject *
    PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
    {
        // 当前线程状态
        PyThreadState *tstate = PyThreadState_GET();
        // 解释器对象去执行frame
        return tstate->interp->eval_frame(f, throwflag);
    }


而interp->eval_frame函数是指向(默认)_PyEval_EvalFrameDefault

.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {

        这里就是具体执行字节码的地方
        
    }

执行字节码
==============

通过dis查到这个操作码是STORE_SUBSCR:

.. code-block:: python

    In [13]: import dis
    
    In [14]: dis.dis("x[1]='a'")
      1           0 LOAD_CONST               0 ('a')
                  2 LOAD_NAME                0 (x)
                  4 LOAD_CONST               1 (1)
                  6 STORE_SUBSCR
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE

然后在_PyEval_EvalFrameDefault中:

.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 省略代码
        TARGET(STORE_SUBSCR) {
            PyObject *sub = TOP();
            PyObject *container = SECOND();
            PyObject *v = THIRD();
            int err;
            STACKADJ(-3);
            /* container[sub] = v */
            err = PyObject_SetItem(container, sub, v);
            Py_DECREF(v);
            Py_DECREF(container);
            Py_DECREF(sub);
            if (err != 0)
                goto error;
            DISPATCH();
        }
        // 省略代码
    }


执行中调用的接口不是具体的实现, 而是一个通用的接口, 比如PyObject_SetItem, 这个接口负责根据对象不同调用不同的实现.


中间层接口
================

中间层的接口放在cpython/Objects/abstract.c中, 比如上面的PyObject_SetItem:

.. code-block:: c


    int
    PyObject_SetItem(PyObject *o, PyObject *key, PyObject *value)
    {
        PyMappingMethods *m;
    
        if (o == NULL || key == NULL || value == NULL) {
            null_error();
            return -1;
        }
        // 先判断对象是否定义有mapping的操作
        m = o->ob_type->tp_as_mapping;
        if (m && m->mp_ass_subscript)
            return m->mp_ass_subscript(o, key, value);
    
        // 再判断对象是否定义有sequence的操作
        if (o->ob_type->tp_as_sequence) {
            if (PyIndex_Check(key)) {
                Py_ssize_t key_value;
                key_value = PyNumber_AsSsize_t(key, PyExc_IndexError);
                if (key_value == -1 && PyErr_Occurred())
                    return -1;
                return PySequence_SetItem(o, key_value, value);
            }
            else if (o->ob_type->tp_as_sequence->sq_ass_item) {
                type_error("sequence index must be "
                           "integer, not '%.200s'", key);
                return -1;
            }
        }
        
        // 没有mapping操作, 也没定义有sequence操作, 报错
        type_error("'%.200s' object does not support item assignment", o);
        return -1;
    }

所以, 这一层只是负责调用对象对应的方法而已, 具体实现交给对象本身



对象层/gc/内存管理
====================


负责实现具体的操作, 比如上面的PyObject_SetItem, 在dict对象中, 有:



.. code-block:: c

    // 这里定义了mapping操作
    PyTypeObject PyDict_Type = {
        &dict_as_mapping,                           /* tp_as_mapping */
    }
    
    // mapping的实现
    static PyMappingMethods dict_as_mapping = {
        (lenfunc)dict_length, /*mp_length*/
        (binaryfunc)dict_subscript, /*mp_subscript*/
        // 这个就是set_item的函数
        (objobjargproc)dict_ass_sub, /*mp_ass_subscript*/
    };


并且, 对象实现的时候是需要跟gc和内存管理交互的:

1. 如果对象是需要gc的对象, 那么new一个对象的时候会把新建的对象加入到gc链表中.

2. new一个对象的时候, 往往有自己的缓存, 需要自己实现, 否则直接通过内存管理接口去分配内存.

