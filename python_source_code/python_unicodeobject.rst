unicode
==========

1. python3中, str和unicode合并了, 只存在unicode

2. unicode的缓存(intern), is操作, intern手动缓存, 编译缓存和优化

3. 缓存长度为20其实是, 使用\*重复20次以上的, python不会编译时求值, 而是运行时求值

3. find算法(BM和Horspool的综合)


缓存参考: http://guilload.com/python-string-interning/


----

PyUnicodeObject
==================



缓存机制
===================

is操作的区别

.. code-block:: python

    In [124]: x='foo!'
    
    In [125]: y='foo!'
    
    In [126]: x is y
    Out[126]: False
    
    In [127]: x='awd'
    
    In [128]: y='awd'
    
    In [129]: x is y
    Out[129]: True


在编译中看看foo!和awd的区别, **每一个语句都会编译成一个codeobject, 每一个codeobject都有自己的consts常量**, 然后其中常量会保存在codeobject.consts中


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
        
        // 这里回去操作consts
        intern_string_constants(consts);
        
        // 省略代码
    
    }

而intern_string_constants会去根据一定的规则去缓存unicode


cpython/Objects/codeobject.c

.. code-block:: c

    static int
    intern_string_constants(PyObject *tuple)
    {
        // 省略代码

        // all_name_chars则是缓存的判断条件
        if (all_name_chars(v)) {
          // 省略代码
          PyUnicode_InternInPlace(&v);
        }
        // 省略代码
    }

all_name_chars
=================

all_name_chars是判断一个字符是否需要缓存的地方

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
    
        // 这里是初始化过程, 所有的NAME_CHARS的字符, 在ok_name_char中都需要置1
        if (ok_name_char[*name_chars] == 0) {
            const unsigned char *p;
            for (p = name_chars; *p; p++)
                ok_name_char[*p] = 1;
        }
        s = PyUnicode_1BYTE_DATA(o);
        e = s + PyUnicode_GET_LENGTH(o);
        // 下面的循环会一个字符一个字符去判断是否
        // 是常规字符
        // e是最后一个字符, s是从第一个字符开始
        while (s != e) {
            // *s就是当前位置的字符
            if (ok_name_char[*s++] == 0)
                return 0;
        }
        return 1;
    }

其中判断流程是:

1. 非ascii字符不缓存, 比如'abc我'带有中文就不会缓存了

2. 创建长度为256的数组ok_name_char

3. 拿到常规字符串NAME_CHARS, 也就是0-9, 26个字母的大小写,也就是所有的长度为0和1的字符都默认被缓存掉了

4. 然后把NAME_CHARS在ok_name_char的位置设置为1

5. 逐个字符去判断是否是常规字符, 也就是其ok_name_char中缓存位是否是１, 如果是0, 退出

所以:

1. 常规字符默认是缓存的, 0-9和26个字母大小写

2. 除了常规字符之外都是特殊字符, 都不会缓存, 包括unicode字符

3. all_name_chars中没有长度判断, 所以

.. code-block:: python

    In [1]: x='aaaaaaaaaaaaaaaaaaaa'
    
    In [2]: y='aaaaaaaaaaaaaaaaaaaa'
    
    In [3]: x is y
    Out[3]: True
    
    In [4]: z='aaaaaaaaaaaaaaaaaaaa'
    
    In [5]: x is z
    Out[5]: True
    
    In [6]: x='aaaaaaaaaaaaaaaaaaaab'
    
    In [7]: y='aaaaaaaaaaaaaaaaaaaab'
    
    In [8]: x is y
    Out[8]: True


第一次的x, y, z赋值是20个a, 然后第二次的x, y赋值是20个a加上一个b, 一共21的长度

但是, 为什么:

.. code-block:: python

    In [16]: x='a' * 20
    
    In [17]: y='a' * 20
    
    In [18]: x is y
    Out[18]: True
    
    In [19]: y='a' * 21
    
    In [20]: x='a' * 21
    
    In [21]: x is y
    Out[21]: False

**看下面的长度部分**


PyUnicode_InternInPlace
========================

而PyUnicode_InternInPlace则是处理缓存的具体函数


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
        // 判断的依据是PyUnicodeObject->state.interned是否是SSTATE_INTERNED_MORTAL, 也就是1
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
        // -----------注意, 这里是interned中的t不等于s
        // -----------那么把p指针指向的unicode指向interned中的t
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

**缓存的时候, 会指向同一个对象**

foo!这个字符串:

1. 一开始ok_name_char是256空数组
   
2. 然后经过第一个循环之后, ok_name_char赋值了, 比如f这个字符的ascii数值是102, 也就是 *ok_name_char[102] = 1*

3. 然后逐个循环foo!, 循环到!这个字符的时候, 发现!的ascii值是33, 并且ok_name_char[33] == 0, 所以返回0, 不缓存

awd这个字符串

1. 第一个x='awd', ok_name_char返回1, 所以调用PyUnicode_InternInPlace, 缓存了awd

2. 第二个y='awd', ok_name_char返回1, 所以调用PyUnicode_InternInPlace去缓存awd

3. **而PyUnicode_InternInPlace的作用是会把y指向x指向的awd**

4. 所以is操作返回True

那么在函数中呢?

函数中的consts
==================

.. code-block:: python

    In [137]: def test():
         ...:     a = 'foo!'
         ...:     b = 'foo!'
         ...:     print(a is b)
         ...:     return
         ...: 
    
    In [137]: test()
    True

按照之前的说法, foo!不满足缓存条件, 那么a, b调用is操作应该是不同的呀, 为什么会相同呢?


先看看dis的结果

.. code-block:: python

    In [138]: dis.dis(test)
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
    

看到带有LOAD_CONST语句, 看看LOAD_CONST是干嘛的


.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 省略了很多代码

        PyObject *consts;

        // 拿到先当前的frame对象
        tstate->frame = f;

        // 拿到co_consts对象
        consts = co->co_consts;

            PREDICTED(LOAD_CONST);
            TARGET(LOAD_CONST) {
                // 从consts拿到对象
                PyObject *value = GETITEM(consts, oparg);
                Py_INCREF(value);
                PUSH(value);
                FAST_DISPATCH();
            }
    }

然后我们看看test函数的codeobject的consts属性

.. code-block:: python

    In [139]: x=test.__code__
    
    In [140]: x
    Out[140]: <code object test at 0x7f1847bebed0, file "<ipython-input-136-22e7d50e9716>", line 1>
    
    In [141]: x.co_consts
    Out[141]: (None, 'foo!')

我们看到, consts包含了一个foo!字符串(虽然我们赋值了两次, 但是只有一个), 然后保存到consts中, 所以当LOAD_CONST执行的时候,

从consts拿到的是同一个foo!

**所以, 虽然foo!没有被缓存掉(intern), 但是由于codeobject中只存储了一个foo!, 但是LOAD_CONST拿到的是同一个对象, 搜易is返回True**

**这里跟缓存没关系, 只是说函数中拿到的foo!是同一个.**

运行时的缓存(编译优化)
==============================

python会在编译成字节码的时候把常量和常量计算的结果给缓存掉, 带计算的, 运行时计算的结果并不会的, 比如x='a', y='b', c = x + y, c就是运行时计算的.

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


intern手动缓存
====================


sys.intern调用到PyUnicode_InternInPlace, 手动缓存指定的字符

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

字符串的长度和缓存
====================

在all_name_chars那部分中, 针对长度的例子:


.. code-block:: python

    In [1]: x='a' * 20
    
    In [2]: y='a' * 20
    
    In [3]: x is y
    Out[3]: True
    
    In [4]: x='a' * 21
    
    In [5]: y='a' * 21
    
    In [6]: x is y
    Out[6]: False
    
    In [7]: x='aaaaaaaaaaaaaaaaaaaab'
    
    In [8]: y='aaaaaaaaaaaaaaaaaaaab'
    
    In [9]: len(x), len(y)
    Out[9]: (21, 21)
    
    In [10]: x is y
    Out[10]: True

使用\*操作符号, 产生长度21的字符串是不能缓存的, 但是使用常量的21个字符串是可以缓存的, 为什么呢?

看看dis结果:

.. code-block:: c

    In [12]: dis.dis("x='a' * 20")
      1           0 LOAD_CONST               3 ('aaaaaaaaaaaaaaaaaaaa')
                  2 STORE_NAME               0 (x)
                  4 LOAD_CONST               2 (None)
                  6 RETURN_VALUE
    
    In [13]: dis.dis("x='a' * 21")
      1           0 LOAD_CONST               0 ('a')
                  2 LOAD_CONST               1 (21)
                  4 BINARY_MULTIPLY
                  6 STORE_NAME               0 (x)
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE
    
    In [14]: dis.dis("x='aaaaaaaaaaaaaaaaaaaab'")
      1           0 LOAD_CONST               0 ('aaaaaaaaaaaaaaaaaaaab')
                  2 STORE_NAME               0 (x)
                  4 LOAD_CONST               1 (None)
                  6 RETURN_VALUE

看起来, \*重复20以上次数的操作, 不是拿常量了, 而是运行时计算, 所以, x='a'\*21不是编译时候求值, 所以不会走consts缓存的流程




整数的intern
==================

整数也可以intern的, 但是整数的intern应该是和小整数内存池有关.



字符串查找
===============

http://www.laurentluce.com/posts/python-string-objects-implementation/


查找算法参考了BM算法和Horspool算法

