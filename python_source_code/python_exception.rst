##########
python异常
##########

内建异常的结构
===================

先来看看KeyError是在哪里初始化的

在python解释器初始化的时候, 会去初始化各个module, 其中cpython/Python/pylifecycle.c, 函数_Py_InitializeEx_Private

则是负责加载和初始化module的, 其中会调用函数_PyExc_Init, 这个函数就是初始化内建异常对象(其实可以称为类)

cpython/Objects/exception.c

.. code-block:: c


    void
    _PyExc_Init(PyObject *bltinmod)
    {
        PyObject *bdict;
    
        // 前面还有很多异常

        PRE_INIT(KeyError)

        // 后面还有很多

        // 拿到内建字典
        bdict = PyModule_GetDict(bltinmod);
        if (bdict == NULL)
            Py_FatalError("exceptions bootstrapping error.");

        // 然后POST_INIT

        // 前面还有很多异常

        // 设置到内建字典中
        POST_INIT(KeyError)

        // 后面还有很多
    
    
    }


而这个KeyError则不是直接去创建的, 而是调用函数去创建的, 这个函数是ComplexExtendsException

先看看ComplexExtendsException的定义

cpython/Objects/exception.c

.. code-block:: c

    #define ComplexExtendsException(EXCBASE, EXCNAME, EXCSTORE, EXCNEW, \
                                    EXCMETHODS, EXCMEMBERS, EXCGETSET, \
                                    EXCSTR, EXCDOC) \
    static PyTypeObject _PyExc_ ## EXCNAME = { \
        PyVarObject_HEAD_INIT(NULL, 0) \
        # EXCNAME, \
        sizeof(Py ## EXCSTORE ## Object), 0, \
        (destructor)EXCSTORE ## _dealloc, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, \
        (reprfunc)EXCSTR, 0, 0, 0, \
        Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE | Py_TPFLAGS_HAVE_GC, \
        PyDoc_STR(EXCDOC), (traverseproc)EXCSTORE ## _traverse, \
        (inquiry)EXCSTORE ## _clear, 0, 0, 0, 0, EXCMETHODS, \
        EXCMEMBERS, EXCGETSET, &_ ## EXCBASE, \
        0, 0, 0, offsetof(Py ## EXCSTORE ## Object, dict), \
        (initproc)EXCSTORE ## _init, 0, EXCNEW,\
    }; \

看起来很难看懂呀, 其实只需要把这个函数当成一个动态创建对象的函数, 比如

第二个参数是EXCNAME, 然后会创建这样一个PyTypeObject, 其前缀是\_PyExc\_, 然后后面跟EXCNAME(就是##后面)

所以这个函数就是说, 根据传入的EXCNAME, EXCBASE等等函数动态创建一个前缀为\_PyExc\_的异常对象

然后看看那KeyError的定义地方:

.. code-block:: c

    static PyObject *
    KeyError_str(PyBaseExceptionObject *self)
    {
        /* If args is a tuple of exactly one item, apply repr to args[0].
           This is done so that e.g. the exception raised by {}[''] prints
             KeyError: ''
           rather than the confusing
             KeyError
           alone.  The downside is that if KeyError is raised with an explanatory
           string, that string will be displayed in quotes.  Too bad.
           If args is anything else, use the default BaseException__str__().
        */
        if (PyTuple_GET_SIZE(self->args) == 1) {
            return PyObject_Repr(PyTuple_GET_ITEM(self->args, 0));
        }
        return BaseException_str(self);
    }

    ComplexExtendsException(PyExc_LookupError, KeyError, BaseException,
                        0, 0, 0, 0, KeyError_str, "Mapping key not found.");


**其实就是动态创建了_PyExec_KeyError这类型**

KeyError是_PyExec_KeyError类型的类(PyTypeObject对象), 继承于PyExc_LookupError, 然后KeyError类型(PyTypeObject)的名称tp_name就是KeyError等等

所以就是

1. _PyExec_KeyError也是PyTypeObject, 只是里面的属性和其他的PyTypeObject不同, 其中ob_type=PyType_Type

2. 然后创建一个_PyExec_KeyError类型的对象的话, 也就是PyObject, 那么该PyObject的ob_type就是_PyExec_KeyError
   
3. PyType_Type就是内建的type类型, 所有的对象都继承于它, 包括类型, 这里理解一下python中类是类型, 类型也是类的概念

POST_INIT
============

PRE_INIT函数没什么工作, 调用PyType_Ready去检查这个类是否就绪了, 先不看

然后, POST_INIT就是设置异常名称对应异常类(对象)到内建字典中的

POST_INIT的定义:

.. code-block:: c

    #define POST_INIT(TYPE) \
        if (PyDict_SetItemString(bdict, # TYPE, PyExc_ ## TYPE)) \
            Py_FatalError("Module dictionary insertion problem.");

也就是说, 传入内建字典(bdict, 之前我们拿到的), 和TYPE, 也就是名称, 以及前缀为PyExc\_ ##TYPE的对象, 设置{Type: _PyExc_## TYPE}这样的关系

**注意的是, 这里看起来是传入的是PyExec_## TYPE, 但是其实传入到函数的话是_PyExec_ ## TYPE, 暂时不太懂为什么, C语言比较渣**

然后 *POST_INIT(KeyError)** 就是, 调用到PyDict_SetItemString

.. code-block:: c

    int
    PyDict_SetItemString(PyObject *v, const char *key, PyObject *item)
    {
        PyObject *kv;
        int err;
        kv = PyUnicode_FromString(key);
        if (kv == NULL)
            return -1;
        PyUnicode_InternInPlace(&kv); /* XXX Should we really? */
        err = PyDict_SetItem(v, kv, item);
        Py_DECREF(kv);
        return err;
    }


其中, 传入的参数, 第一个就是内建字典, 第二个就是名称, 然后就是字符串"KeyError", 第三个就是_PyExec_KeyError, 这样

**内建的异常类型和异常名称就在内建字典中对应起来了**

**注意, 为什么传入的是KeyError, 一个变量形式, 而不是字符串形式"KeyError", 然后在PyDict_SetItemString中收到就是字符串"KeyError", 这个不清楚(C语言比较渣)**


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


raise异常
==============

先看看dis出来的raise语句的opcode

.. code-block:: python


    In [5]: dis.dis("raise KeyError")
      1           0 LOAD_NAME                0 (KeyError)
                  2 RAISE_VARARGS            1
                  4 LOAD_CONST               0 (None)
                  6 RETURN_VALUE

我们知道, LOAD_NAME会先找local, 再找global, 最后找builtin, 所以, 最后会从内建字典中找到KeyError, 也就是之前

说的_PyExec_KeyError对象, 然后在RAISE_VARARGS中, 最终还是回调用到PyErr_Format去设置当前tstate的异常变量的

.. code-block:: c

        TARGET(RAISE_VARARGS) {
            PyObject *cause = NULL, *exc = NULL;
            switch (oparg) {
            case 2:
                cause = POP(); /* cause */
                /* fall through */
            case 1:
                exc = POP(); /* exc */
                /* fall through */
            case 0:
                if (do_raise(exc, cause)) {
                    why = WHY_EXCEPTION;
                    goto fast_block_end;
                }
                break;
            default:
                PyErr_SetString(PyExc_SystemError,
                           "bad RAISE_VARARGS oparg");
                break;
            }
            goto error;
        }

**然后在C语言中, switch如果你不主动break的话, 会一直走case, 直到走到一个分支带有break才退出**

所以, 当你 *raise KeyError* 的时候, 先走 *case 1*, 执行POP操作, 拿到异常, 然后走到 *case 0*, 执行do_raise函数


do_raise
==========

do_raise函数的话, 就是

1. 所我们可以raise KeyError, 引发异常类

   或者raise KeyError('abc'), 引发异常实例

   否则报错

2. 生成异常类型的实例对象, value->ob_type=_PyExec_KeyError

3. 调用PyErr_SetObject去设置tstate->curexc_type, 从而引发异常



.. code-block:: c

    /* Logic for the raise statement (too complicated for inlining).
       This *consumes* a reference count to each of its arguments. */
    static int
    do_raise(PyObject *exc, PyObject *cause)
    {
        PyObject *type = NULL, *value = NULL;
    
        // 如果传入的exc的是NULL
        // 但是raise的话一般都不是NULL
        // 所以代码先不看
        if (exc == NULL) {
        }
    
        // 如果我们是raise KeyError
        // 那么就是异常类
        // 走下面这个if
        if (PyExceptionClass_Check(exc)) {
            type = exc;
            // 这里会调用到ob_type->tp_call
            // 生成异常对象实例
            value = PyObject_CallObject(exc, NULL);

            // 生成value出错
            if (value == NULL)
                goto raise_error;

            // 生成value出错
            if (!PyExceptionInstance_Check(value)) {
                // 报异常, 异常是PyExc_TypeError
                PyErr_Format(PyExc_TypeError,
                             "calling %R should have returned an instance of "
                             "BaseException, not %R",
                             type, Py_TYPE(value));
                goto raise_error;
            }
        }
        // 如果我们是raise KeyError('abc')
        // 那么传入的exec就是异常实例了
        // 走下面这个if
        else if (PyExceptionInstance_Check(exc)) {
            value = exc;
            type = PyExceptionInstance_Class(exc);
            Py_INCREF(type);
        }
        // 这个else就是说
        // raise后面必须是异常或者异常类
        // 否则报错
        else {
            /* Not something you can raise.  You get an exception
               anyway, just not what you specified :-) */
            Py_DECREF(exc);
            PyErr_SetString(PyExc_TypeError,
                            "exceptions must derive from BaseException");
            goto raise_error;
        }

        // 当我们直接raise KeyError的时候
        // cause一般是NULL, 那么代码先不看
        if (cause) {

        }

        // 最后调用PyErr_SetObject去设置
        // tstate中的curexc_type
        PyErr_SetObject(type, value);
        /* PyErr_SetObject incref's its arguments */
        Py_XDECREF(value);
        Py_XDECREF(type);
        return 0;
    
    }

其中, 如果raise的是异常类的话, 那么调用到ob_type->tp_call函数

_PyExc_KeyError的ob_type是PyType_Type, 也就是通用类型对象, 然后继承自BaseException, 所以, 调用当调用PyObject_CallObject, 传入参数_PyExc_KeyError

的时候, 在python中, 就是type object也是一个callable对象, 其call方法就是tp_call, 而_PyExc_KeyError中由于继承自BaseException, 那么PyObject_CallObject

函数会调用传入参数的ob_type->tp_call, 其中, tp_call会调用tp_new去生成一个新实例, 也就是调用到BaseException中的tp_new

也就是BaseException_new, 其返回一个PyBaseExceptionObject

.. code-block:: c

    static PyObject *
    BaseException_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
        // 会返回self, 也就是PyBaseExceptionObject对象
        PyBaseExceptionObject *self;
    
        self = (PyBaseExceptionObject *)type->tp_alloc(type, 0);
        if (!self)
            return NULL;
        /* the dict is created on the fly in PyObject_GenericSetAttr */
        self->dict = NULL;
        self->traceback = self->cause = self->context = NULL;
        self->suppress_context = 0;
    
        if (args) {
            self->args = args;
            Py_INCREF(args);
            return (PyObject *)self;
        }
    
        self->args = PyTuple_New(0);
        if (!self->args) {
            Py_DECREF(self);
            return NULL;
        }
    
        return (PyObject *)self;
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

try/except
===============

基本上就是字节码之间跳来跳去的

先看看try/except的字节码:

.. code-block:: python

    In [1]: def test():
       ...:     try:
       ...:         print(a)
       ...:     except NameError:
       ...:         print('no a')
       ...:     return
       ...: 
    
    In [2]: import dis
    
    In [3]: dis.dis(test)
      2           0 SETUP_EXCEPT            12 (to 14)
    
      3           2 LOAD_GLOBAL              0 (print)
                  4 LOAD_GLOBAL              1 (a)
                  6 CALL_FUNCTION            1
                  8 POP_TOP
                 10 POP_BLOCK
                 12 JUMP_FORWARD            28 (to 42)
    
      4     >>   14 DUP_TOP
                 16 LOAD_GLOBAL              2 (NameError)
                 18 COMPARE_OP              10 (exception match)
                 20 POP_JUMP_IF_FALSE       40
                 22 POP_TOP
                 24 POP_TOP
                 26 POP_TOP
    
      5          28 LOAD_GLOBAL              0 (print)
                 30 LOAD_CONST               1 ('no a')
                 32 CALL_FUNCTION            1
                 34 POP_TOP
                 36 POP_EXCEPT
                 38 JUMP_FORWARD             2 (to 42)
            >>   40 END_FINALLY
    
      6     >>   42 LOAD_CONST               0 (None)
                 44 RETURN_VALUE

1. 从字节码中, 我们可以看到, SETUP_EXCEPT是遇到了try关键字, 后面跟的14则是如果出现异常

   需要跳到哪一行字节码处理, 在例子中, 如果出现异常, 那么会走到14开头的字节码流程

   如果没有出现异常, 那么会走到12, 一个跳转字节码, 会跳转到42

   其中14这个字节码地址是需要保存的, 保存到当前frame中

2. 14是DUP_TOP, 所以我们可知, 异常会放到栈顶, 然后14就DUP_TOP获得异常(注意, 不是POP_TOP, 只是获取栈顶而已), 接着16, 18是判断异常是否是我们except的

   20则是获得18的判断, 如果异常是我们except的, 那么一直走, 走到28开始执行except里面的代码块, 然后一直执行到38, 会跳到42
   
   如果不是我们捕获的异常, 那么跳到40

3. 16到36是我们在execpt中的代码块, 走完之后需要跳出try/execpt代码块, 所以, 38则是一个跳转, 跳转到42

   42是在40这个END_FINALLY字节码之后, 显然, 是try/except之后的代码

3. 40只是一个END_FINALLY, 也就是出现异常, 但是不是我们捕获的异常, 所以这个字节码就是设置异常然后退出的操作

4. 如果定义了finally, 那么无论try/except出现什么情况, 最后都会跳转到finally中定义的字节码, 然后再顺序执行就好了


.. code-block:: python

    In [8]: def test():
       ...:     try:
       ...:         print(pq)
       ...:     except KeyError:
       ...:         print('ke')
       ...:     finally:
       ...:         print('finally')
       ...:     k = 1
       ...:     return k
       ...: 
    
    In [9]: dis.dis(test)
      2           0 SETUP_FINALLY           46 (to 48)
                  2 SETUP_EXCEPT            12 (to 16)
    
      3           4 LOAD_GLOBAL              0 (print)
                  6 LOAD_GLOBAL              1 (pq)
                  8 CALL_FUNCTION            1
                 10 POP_TOP
                 12 POP_BLOCK
                 14 JUMP_FORWARD            28 (to 44)
    
      4     >>   16 DUP_TOP
                 18 LOAD_GLOBAL              2 (KeyError)
                 20 COMPARE_OP              10 (exception match)
                 22 POP_JUMP_IF_FALSE       42
                 24 POP_TOP
                 26 POP_TOP
                 28 POP_TOP
    
      5          30 LOAD_GLOBAL              0 (print)
                 32 LOAD_CONST               1 ('ke')
                 34 CALL_FUNCTION            1
                 36 POP_TOP
                 38 POP_EXCEPT
                 40 JUMP_FORWARD             2 (to 44)
            >>   42 END_FINALLY
            >>   44 POP_BLOCK
                 46 LOAD_CONST               0 (None)
    
      7     >>   48 LOAD_GLOBAL              0 (print)
                 50 LOAD_CONST               2 ('finally')
                 52 CALL_FUNCTION            1
                 54 POP_TOP
                 56 END_FINALLY
    
      8          58 LOAD_CONST               3 (1)
                 60 STORE_FAST               0 (k)
    
      9          62 LOAD_FAST                0 (k)
                 64 RETURN_VALUE


显然, 记住了finally和try的handler字节码的地址, finally是48, try则是16

所以, 无论如何, try/except要么跳到42(地址为22的字节码), 说明发生的异常不是我们需要捕获的, 抛出异常, 然后顺序执行到finally(48)

要么跳转到44(地址是40的字节码, 也就是except里面的代码), 顺序执行到finally(地址是48)

看看具体的C代码:

SETUP_EXCEPT/SETUP_FINALLY
--------------------------------

这个是try/finally语句, 保存处理异常的handler, 也就是遇到异常, 去哪个字节码开始处理流程

.. code-block:: c

        TARGET(SETUP_LOOP)
        TARGET(SETUP_EXCEPT)
        TARGET(SETUP_FINALLY) {
            /* NOTE: If you add any new block-setup opcodes that
               are not try/except/finally handlers, you may need
               to update the PyGen_NeedsFinalizing() function.
               */

            // 这个函数就是保存异常处理的字节码的地方
            PyFrame_BlockSetup(f, opcode, INSTR_OFFSET() + oparg,
                               STACK_LEVEL());
            DISPATCH();
        }


1. 其中, INSTR_OFFSET + oparg就是得到处理异常的字节码的地址了, 在上面的例子中就是14

2. PyFrame_BlockSetup就是保存所谓的exception handler了, 也就是地址为14的字节码


.. code-block:: c

    void
    PyFrame_BlockSetup(PyFrameObject *f, int type, int handler, int level)
    {
        PyTryBlock *b;
        if (f->f_iblock >= CO_MAXBLOCKS)
            Py_FatalError("XXX block stack overflow");
        b = &f->f_blockstack[f->f_iblock++];
        b->b_type = type;
        b->b_level = level;
        // 传入的14就是handler
        b->b_handler = handler;
    }

注意, 保存的handler是存入数组的, 注意看b的赋值. 所以计算try和finally都保存handler到frame, 但是不会覆盖


error/exception handler
=========================

虽然是叫exception handler, 其实只是异常处理的字节码地址, 跳转到该地址继续执行, 一般来说就是比较异常和except后面跟的异常

一般, 当字节码出错的时候, 会走到error代码块, error代码块中会判断当期那frame中是否保存了exception handler, 如果有, 则跳转

error代码块

.. code-block:: c

    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
    
    error:
    
            assert(why == WHY_NOT);
            why = WHY_EXCEPTION;
    
    
    fast_block_end:
            assert(why != WHY_NOT);
    
            /* Unwind stacks if a (pseudo) exception occurred */
            while (why != WHY_NOT && f->f_iblock > 0) {
                /* Peek at the current block. */
                PyTryBlock *b = &f->f_blockstack[f->f_iblock - 1];
    
                assert(why != WHY_YIELD);
                if (b->b_type == SETUP_LOOP && why == WHY_CONTINUE) {
                    why = WHY_NOT;
                    JUMPTO(PyLong_AS_LONG(retval));
                    Py_DECREF(retval);
                    break;
                }
                /* Now we have to pop the block. */
                f->f_iblock--;
    
                if (b->b_type == EXCEPT_HANDLER) {
                    UNWIND_EXCEPT_HANDLER(b);
                    continue;
                }
                UNWIND_BLOCK(b);
                if (b->b_type == SETUP_LOOP && why == WHY_BREAK) {
                    why = WHY_NOT;
                    JUMPTO(b->b_handler);
                    break;
                }
                // 如果是出现异常, 并且是try/except代码块, 也即是block type(b_type)
                // 是SETUP_EXCEPT
                if (why == WHY_EXCEPTION && (b->b_type == SETUP_EXCEPT
                    || b->b_type == SETUP_FINALLY)) {
                    PyObject *exc, *val, *tb;

                    // 找到handler
                    int handler = b->b_handler;

                    PUSH(tb);
                    PUSH(val);
                    PUSH(exc);
                    why = WHY_NOT;
                    // 跳转到handler
                    JUMPTO(handler);
                    break;
                }
                // 如果是finally字节码
                // 那么跳转到handler的字节码
                if (b->b_type == SETUP_FINALLY) {
                    if (why & (WHY_RETURN | WHY_CONTINUE))
                        PUSH(retval);
                    PUSH(PyLong_FromLong((long)why));
                    why = WHY_NOT;
                    JUMPTO(b->b_handler);
                    break;
                }
            } /* unwind stack */
    
            /* End the loop if we still have an error (or return) */
    
            if (why != WHY_NOT)
                break;
    
            assert(!PyErr_Occurred());
    
        } /* main loop */
    
    }

1. error代码块会判断, 如果是处于try/except(b_type, block_type, 是SETUP_EXCEPT)

   那么会跳转到handler

2. 同样, 如果是处于finally代码块(SETUP_FINALLY), 那么同样会跳到handler字节码处


