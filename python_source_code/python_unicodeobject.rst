unicode
==========

1. python3中, str和unicode合并了, 只存在unicode

2. inter缓存机制

3. find算法(BM和Horspool的综合)


----

PyUnicodeObject
==================



sys.intern手动缓存
===================

参考: http://guilload.com/python-string-interning/

简单来说, intern会把字符串当成全局唯一一个, 然后可以减少内存分配.

cpython/Python/sysmodule.c

.. code-block:: c

    static PyObject *
    sys_intern(PyObject *self, PyObject *args)
    {
        // s是解析传入参数之后的对象
        PyObject *s;
        if (!PyArg_ParseTuple(args, "U:intern", &s))
            return NULL;
        if (PyUnicode_CheckExact(s)) {
            Py_INCREF(s);
            // 这里调用一下
            PyUnicode_InternInPlace(&s);
            // 然后返回
            return s;
        }
        else {
            PyErr_Format(PyExc_TypeError,
                            "can't intern %.400s", s->ob_type->tp_name);
            return NULL;
        }
    }

PyUnicode_InternInPlace
==========================

这里是intern的流程

.. code-block:: c

    void
    PyUnicode_InternInPlace(PyObject **p)
    {
        PyObject *s = *p;
        PyObject *t;
        // 下面是一顿判断s(也就是p)是不是unicode
    #ifdef Py_DEBUG
        assert(s != NULL);
        assert(_PyUnicode_CHECK(s));
    #else
        if (s == NULL || !PyUnicode_Check(s))
            return;
    #endif
        /* If it's a subclass, we don't really know what putting
           it in the interned dict might do. */
        if (!PyUnicode_CheckExact(s))
            return;
        // 这个是判断s是否已经被intern了
        // 判断的依据是s对象里面的state.interned是否是SSTATE_INTERNED_MORTAL
        // 继续看下嘛
        if (PyUnicode_CHECK_INTERNED(s))
            return;
        if (interned == NULL) {
            // 这里是初始化interned字典的地方
            interned = PyDict_New();
            if (interned == NULL) {
                PyErr_Clear(); /* Don't leave an exception */
                return;
            }
        }
        // 调用PyDict_SetDefault设置interned字典
        // 返回的是interned中s的值
        //　因为是setdefault操作, 所以如果s已经被赋值过了, 则返回
        // interned中的s的值
        Py_ALLOW_RECURSION
        t = PyDict_SetDefault(interned, s, s);
        Py_END_ALLOW_RECURSION
        if (t == NULL) {
            PyErr_Clear();
            return;
        }
        // 注意, 这里是interned中的t不等于s
        // 那么把p指针指向的unicode指向interned中的t
        if (t != s) {
            Py_INCREF(t);
            Py_SETREF(*p, t);
            return;
        }
        /* The two references in interned are not counted by refcnt.
           The deallocator will take care of this */
        Py_REFCNT(s) -= 2;
        _PyUnicode_STATE(s).interned = SSTATE_INTERNED_MORTAL;
    }


默认的, 所有的长度为0和1的字符都默认被缓存掉了, 比如常用的0-9, 26个字母的大小写

cpython/Python/codeobject.c

.. code-block:: c

    #define NAME_CHARS \
        "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"

编译时缓存
==============

python会在编译成字节码的时候把常量和常量计算的结果给intern掉, 带计算的, 运行时计算的结果并不会的, 比如x='a', y='b', c = x + y, c就是运行时计算的.

在交互模式下, 比如ipython, 有

.. code-block:: python

    In [47]: p='foo!'
    
    In [48]: q='foo!'
    
    In [49]: p is q
    Out[49]: False

显然, 执行的时候是p和q分两部分去编译的, 所以是先编译执行p, 再编译执行q, 然后由于带了特殊符号, 所以p, q都没有intern(看下一节)

然后如果我们在函数中呢?

.. code-block:: python

    In [50]: def test():
        ...:     a = 'foo!'
        ...:     b = 'foo!'
        ...:     print(a is b)
        ...:     return
        ...: 
    
    In [51]: test()
    True

我们看看dis的结果:

.. code-block:: python

    In [52]: dis.dis(test)
      2           0 LOAD_CONST               1 ('foo!')
                  2 STORE_FAST               0 (a)
    
      3           4 LOAD_CONST               1 ('foo!')
                  6 STORE_FAST               1 (b)
    
      4           8 LOAD_GLOBAL              0 (print)
                 10 LOAD_FAST                0 (a)
                 12 LOAD_FAST                1 (b)
                 14 COMPARE_OP               8 (is)
                 16 CALL_FUNCTION            1
                 18 POP_TOP
    
      5          20 LOAD_CONST               0 (None)
                 22 RETURN_VALUE

显然, 'foo!'这个字符串都是从codeobject的const中拿到的, 而const这个属性是在主键codeobject的时候生成的:

cpython/Objects/codeobject.c

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
    
    intern_string_constants(consts);
    
    // 省略代码

    // 这里把缓存的常量保存到co_consts这个变量中
    co->co_consts = consts;

    // 省略代码
    
    }

而在intern_string_constants中会调用PyUnicode_InternInPlace去intern掉常量

cpython/Objects/codeobject.c

.. code-block:: c

    static int
    intern_string_constants(PyObject *tuple)
    {
        // 省略代码
        if (all_name_chars(v)) {
          // 省略代码
          PyUnicode_InternInPlace(&v);
        }
        // 省略代码
    }

缓存条件是all_name_chars

.. code-block:: c

    #define NAME_CHARS \
        "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"

    /* all_name_chars(s): true iff all chars in s are valid NAME_CHARS */
    static int
    all_name_chars(PyObject *o)
    {
        static char ok_name_char[256];
        static const unsigned char *name_chars = (unsigned char *)NAME_CHARS;
        const unsigned char *s, *e;
    
        // 非ascii码字符串不缓存
        if (!PyUnicode_IS_ASCII(o))
            return 0;
    
        if (ok_name_char[*name_chars] == 0) {
            const unsigned char *p;
            for (p = name_chars; *p; p++)
                ok_name_char[*p] = 1;
        }
        s = PyUnicode_1BYTE_DATA(o);
        e = s + PyUnicode_GET_LENGTH(o);
        while (s != e) {
            if (ok_name_char[*s++] == 0)
                return 0;
        }
        return 1;
    }


所以, 编译的时候会把常量去保存到codeobject的consts属性中, LOAD_CONST会从codeobject中拿到const:


.. code-block:: c

    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    
    {
    
    // 省略很多代码
    
    // 拿到frame上的codeobject
    // co是codeobject结构
    co = f->f_code;
    // 拿到const
    consts = co->co_consts;
    
    // 省略很多代码
    
            TARGET(LOAD_CONST) {
                // 从consts中拿到对象
                PyObject *value = GETITEM(consts, oparg);
                Py_INCREF(value);
                PUSH(value);
                FAST_DISPATCH();
            }
    
    
    // 省略很多代码
    
    }

看起来codeobject中自己保存了consts, 所以consts是针对每一个codeobject的, 但是, 有个问题:


.. code-block:: python

    In [106]: def test():
         ...:     return 'qreg!'
         ...: 
    
    In [107]: 
    
    In [107]: def otest():
         ...:     return 'qreg!'
         ...: 
    
    In [108]: otest() == test()
    Out[108]: True


otest和test明显是两个codeobject, 但是consts中的对象是一样的, 所以

codeobject中的consts中的对象其实在创建的时候拿的就是一个对象了, 比如'qreg'这个字符串已经是被unicode被intern起来了的

所以两个codeobject中的consts指向的对象是同一个!!!



运行时的缓存(编译优化)
==============================

比如

.. code-block:: python

    In [62]: x='a'
    
    In [63]: y='b'
    
    In [64]: x + y is 'ab'
    Out[64]: False
    
    In [65]: 'a' + 'b' is 'ab'
    Out[65]: True

x+y 和 'a' + 'b'的区别就是, 后一句是编译的时候可以直接执行的

而x + y需要在执行的时候(运行到的时候)再计算, 所以不会intern掉, 而是重新生成x+y的结果, 也就是新的'ab'字符串.

**这里就是编译时候的优化了**

比如在函数中

.. code-block:: python

    In [69]: def test():
        ...:     a = 'foo' + 'bar'
        ...:     return a
        ...: 
    
    In [70]: dis.dis(test)
      2           0 LOAD_CONST               3 ('foobar')
                  2 STORE_FAST               0 (a)
    
      3           4 LOAD_FAST                0 (a)
                  6 RETURN_VALUE


可以看到, 在编译的时候, 已经执行了'foo' + 'bar'的计算结果了


特殊符号intern
================

不带特殊符号(!, _等等)的字符串会被intern掉, 带特殊符号的不会.


回到上一节foo!的例子

.. code-block:: python

    In [47]: p='foo!'
    
    In [48]: q='foo!'
    
    In [49]: p is q
    Out[49]: False

但是我们可以有不同的结果:

.. code-block:: python

    In [66]: p = 'awd'
    
    In [67]: q = 'awd'
    
    In [68]: p is q
    Out[68]: True


区别就是foo!中带了特殊符号, 而awd没有.

并且长度是有限制的, 大于20的不会自动intern

.. code-block:: python

    In [71]: 'a' * 20 is 'aaaaaaaaaaaaaaaaaaaa'
    Out[71]: True
    
    In [72]: 'a' * 21 is 'aaaaaaaaaaaaaaaaaaaa'
    Out[72]: False



整数的intern
==================

整数也可以intern的, 但是整数的intern应该是和小整数内存池有关.



字符串查找
===============

http://www.laurentluce.com/posts/python-string-objects-implementation/


查找算法参考了BM算法和Horspool算法

