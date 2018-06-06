############
if的判断流程
############

if判断流程以及对象比较流程


字节码
==========

先看看字节码的执行流程


.. code-block:: python

    In [87]: dis.dis("if x: print('a')")
      1           0 LOAD_NAME                0 (x)
                  2 POP_JUMP_IF_FALSE       12
                  4 LOAD_NAME                1 (print)
                  6 LOAD_CONST               0 ('a')
                  8 CALL_FUNCTION            1
                 10 POP_TOP
            >>   12 LOAD_CONST               1 (None)
                 14 RETURN_VALUE


所以, if语句的字节码是POP_JUMP_IF_FALSE


if的判断
===============

cpython/Python/ceval.c

.. code-block:: c

        TARGET(POP_JUMP_IF_FALSE) {
            PyObject *cond = POP();
            int err;
            if (cond == Py_True) {
                Py_DECREF(cond);
                FAST_DISPATCH();
            }
            if (cond == Py_False) {
                Py_DECREF(cond);
                JUMPTO(oparg);
                FAST_DISPATCH();
            }
            err = PyObject_IsTrue(cond);
            Py_DECREF(cond);
            if (err > 0)
                err = 0;
            else if (err == 0)
                JUMPTO(oparg);
            else
                goto error;
            DISPATCH();
        }

所以, 如果是True/False的话, 直接跳转, 否则进入PyObject_IsTrue

PyObject_IsTrue
=================

.. code-block:: c

    int
    PyObject_IsTrue(PyObject *v)
    {
        Py_ssize_t res;
        if (v == Py_True)
            return 1;
        if (v == Py_False)
            return 0;
        if (v == Py_None)
            return 0;
        else if (v->ob_type->tp_as_number != NULL &&
                 v->ob_type->tp_as_number->nb_bool != NULL)
            res = (*v->ob_type->tp_as_number->nb_bool)(v);
        else if (v->ob_type->tp_as_mapping != NULL &&
                 v->ob_type->tp_as_mapping->mp_length != NULL)
            res = (*v->ob_type->tp_as_mapping->mp_length)(v);
        else if (v->ob_type->tp_as_sequence != NULL &&
                 v->ob_type->tp_as_sequence->sq_length != NULL)
            res = (*v->ob_type->tp_as_sequence->sq_length)(v);
        else
            return 1;
        /* if it is negative, it should be either -1 or -2 */
        return (res > 0) ? 1 : Py_SAFE_DOWNCAST(res, Py_ssize_t, int);
    }


这里直接就是根据不同的类型, 但是大多数是查看对象的长度:

1. 如果是number类型, 调tp_as_number->nbool

2. 如果是字典(mapping)类型, 调用tp_as_mapping->mp_length, 也就是字典的长度

3. 如果是序列(sequence)类型, 调用tp_as_sequence->sq_length, 也就是序列的长度

4. 否则, 直接返回1

5. 如果跳过了4, 那么最后需要校验长度值

那自定义的类型呢?

自定义类型
===========

什么方法都不定义

.. code-block:: python

    class A:
        pass

    a = A()
    if a:
        print('a')

那么直接走上一节的4, 因为a什么方法都没定义, 那么其tp_as_number, tp_as_mapping, tp_as_sequence有结构(也就是不是NULL), 但是

对应的tp_as_number->nb_bool, tp_as_mapping->mp_length和tp_as_sequence->sq_length都没有定义, 所以直接返回1


如果定义了__bool__方法

什么方法都不定义

.. code-block:: python

    class A:
        def __bool__(self):
            return False

    a = A()
    if a:
        print('a')

那么直接走上一节的1, 因为__bool__就是tp_as_number->nbool

如果定义了__len__方法


.. code-block:: python

    class A:
        def __len__(self):
            return 1

    a = A()
    if a:
        print('a')

那么直接走上一节的2, **因为定义了__len__的话, 就是tp_as_sequence->sq_length和tp_as_mapping->mp_length**

但是判断上先判断tp_as_sequence, 所以走2


比较对象
=============

先看看这个问题, 为什么 *x in (x,)* 要快于 *x == x* ?

https://stackoverflow.com/questions/28885132/why-is-x-in-x-faster-than-x-x

其中高票答案总结下来就是:

*Both dispatch to if (left_pointer == right_pointer); the difference is just how much work they do to get there. in just does less.*

in的比较的操作要小于对比操作, 虽然两者都会调用到rich_compare相关的函数


看看字节码

.. code-block:: python

    In [20]: dis.dis("aa == a")
      1           0 LOAD_NAME                0 (aa)
                  2 LOAD_NAME                1 (a)
                  4 COMPARE_OP               2 (==)
                  6 RETURN_VALUE


字节码COMPARE_OP

.. code-block:: c

        TARGET(COMPARE_OP) {
            PyObject *right = POP();
            PyObject *left = TOP();
            PyObject *res = cmp_outcome(oparg, left, right);
            Py_DECREF(left);
            Py_DECREF(right);
            SET_TOP(res);
            if (res == NULL)
                goto error;
            PREDICT(POP_JUMP_IF_FALSE);
            PREDICT(POP_JUMP_IF_TRUE);
            DISPATCH();
        }

所以就是函数cmp_outcome

.. code-block:: c

    static PyObject *
    cmp_outcome(int op, PyObject *v, PyObject *w)
    {
        int res = 0;
        switch (op) {
        case PyCmp_IS:
            res = (v == w);
            break;
        case PyCmp_IS_NOT:
            res = (v != w);
            break;
        case PyCmp_IN:
            res = PySequence_Contains(w, v);
            if (res < 0)
                return NULL;
            break;
        case PyCmp_NOT_IN:
            res = PySequence_Contains(w, v);
            if (res < 0)
                return NULL;
            res = !res;
            break;
        case PyCmp_EXC_MATCH:
            if (PyTuple_Check(w)) {
                Py_ssize_t i, length;
                length = PyTuple_Size(w);
                for (i = 0; i < length; i += 1) {
                    PyObject *exc = PyTuple_GET_ITEM(w, i);
                    if (!PyExceptionClass_Check(exc)) {
                        PyErr_SetString(PyExc_TypeError,
                                        CANNOT_CATCH_MSG);
                        return NULL;
                    }
                }
            }
            else {
                if (!PyExceptionClass_Check(w)) {
                    PyErr_SetString(PyExc_TypeError,
                                    CANNOT_CATCH_MSG);
                    return NULL;
                }
            }
            res = PyErr_GivenExceptionMatches(v, w);
            break;
        default:
            return PyObject_RichCompare(v, w, op);
        }
        v = res ? Py_True : Py_False;
        Py_INCREF(v);
        return v;
    }

所以, 我们看到, in, not in等等操作都是走比对流程的

我们先看看那 *==* 操作, 也就是走default分支

调用了PyObject_RichCompare这个函数, 这个函数就是核心的比对函数

.. code-block:: c

    /* Perform a rich comparison with object result.  This wraps do_richcompare()
       with a check for NULL arguments and a recursion check. */
    
    PyObject *
    PyObject_RichCompare(PyObject *v, PyObject *w, int op)
    {
        PyObject *res;
    
        assert(Py_LT <= op && op <= Py_GE);
        if (v == NULL || w == NULL) {
            if (!PyErr_Occurred())
                PyErr_BadInternalCall();
            return NULL;
        }
        if (Py_EnterRecursiveCall(" in comparison"))
            return NULL;
        res = do_richcompare(v, w, op);
        Py_LeaveRecursiveCall();
        return res;
    }

所以, 也就是函数do_richcompare


.. code-block:: c

    static PyObject *
    do_richcompare(PyObject *v, PyObject *w, int op)
    {
        richcmpfunc f;
        PyObject *res;
        int checked_reverse_op = 0;
    
        // 这里, 
        // 1. 先比较类型, 类型不相等, 走2
        // 2. 比较是否是子类, 是子类走3
        // 3. 如果右参数存在tp_richcompare函数, 走代码块
        if (v->ob_type != w->ob_type &&
            PyType_IsSubtype(w->ob_type, v->ob_type) &&
            (f = w->ob_type->tp_richcompare) != NULL) {
            checked_reverse_op = 1;
            res = (*f)(w, v, _Py_SwappedOp[op]);
            if (res != Py_NotImplemented)
                return res;
            Py_DECREF(res);
        }

        // 这里
        // 如果左参数有tp_richcompare函数, 则调用该函数
        // 如果该函数返回的是未实现, 那么继续, 否则返回
        if ((f = v->ob_type->tp_richcompare) != NULL) {
            res = (*f)(v, w, op);
            if (res != Py_NotImplemented)
                return res;
            Py_DECREF(res);
        }

        // 同样右参数的tp_richcompare函数
        if (!checked_reverse_op && (f = w->ob_type->tp_richcompare) != NULL) {
            res = (*f)(w, v, _Py_SwappedOp[op]);
            if (res != Py_NotImplemented)
                return res;
            Py_DECREF(res);
        }

        // 如果两个对象都没有实现比对方法, 
        // 那么对于==和!=两个操作, 直接比对内存地址

        // 对于其他的操作符, 比如>=, <=等等, 报异常
        /* If neither object implements it, provide a sensible default
           for == and !=, but raise an exception for ordering. */
        switch (op) {
        case Py_EQ:
            res = (v == w) ? Py_True : Py_False;
            break;
        case Py_NE:
            res = (v != w) ? Py_True : Py_False;
            break;
        default:
            PyErr_Format(PyExc_TypeError,
                         "'%s' not supported between instances of '%.100s' and '%.100s'",
                         opstrings[op],
                         v->ob_type->tp_name,
                         w->ob_type->tp_name);
            return NULL;
        }
        Py_INCREF(res);
        return res;
    }


1. 代码中的_Py_SwappedOp是取传入操作符号的相反操作符号

   比如传入的是gt, gt=4, 而_Py_SwappedOp[4] = lt

   所以, 如果传入的是gt, 然后左参数的gt返回的是Py_NotImplemented

   那么调用右参数的相反操作符, 比如要调用左参数的gt, 相应的, 调用

   右参数的lt

2. 关于Py_NotImplemented, 可以看看long类型的例子, long中调用比对的时候, 会校验传入参数是否是long类型

   不是的话, 返回Py_NotImplemented

.. code-block:: c

    // 这个是校验类型的
    // 两个传参有一个不是long类型, 会返回Py_NotImplemented
    #define CHECK_BINOP(v,w)                                \
        do {                                                \
            if (!PyLong_Check(v) || !PyLong_Check(w))       \
                Py_RETURN_NOTIMPLEMENTED;                   \
        } while(0)

    // 这个是long类型的tp_richcompare
    static PyObject *
    long_richcompare(PyObject *self, PyObject *other, int op)
    {
        int result;
        PyObject *v;
        // 校验传参类型, 返回Py_NotImplemented
        CHECK_BINOP(self, other);

        // 后面是具体操作, 先省略

        return v;
    }


3. 关于tp_richcompare, 如果没有定义对比魔术方法(\_\_gt\_\_, __lt__等等), 那么默认是object_richcompare

   否则是slot_tp_richcompare, 而slot_tp_richcompare则是根据操作符去返回指定的方法

   比如操作符是gt, 那么slot_tp_richcompare中, 就有 *func = lookup_method(self, &name_op[op]);*

   也就是返回\_\_gt\_\_方法, 然后调用func

4. 关于object_richcompare, 只针对==和!=两个操作符有指令, 也就是返回两个对象的内存地址是否相等

   其他比对(gt, lt等等), 则返回Py_NotImplemented

比对的流程:

1. 第一个if判断的是: 类型不相等, 并且右参数是左参数的子类, 并且右参数类存在tp_richcompare函数

   则调用右参数的tp_richcompare函数, 传入的操作符是参入的操作符的相反操作符(gt变成lt)

   所以这里主要是判断子类的, 但是调用的是右参数的相反操作符函数

2. 如果1没有返回, 那么接下来就是判断左参数的gt, 和右参数的lt方法

   如果都没有返回, 那么走3

3. 在3中, 主要是针对==和!=操作符, == 和!=会直接比对内存地址, 返回True/False

   而其他操作符(gt, lt)则直接报错
   
例子:

.. code-block:: python

    In [1]: class A:
       ...:     pass
       ...: 
    
    In [2]: class B:
       ...:     def __gt__(self, data):
       ...:         return self.data > data
       ...:     
    
    In [3]: a=A()
    
    In [4]: b=B()
    
    In [5]: b.data = 10

然后我们对比:

.. code-block:: python

    In [6]: a == 1
    Out[6]: False

这里, 走了比对流程的2, 3, 进入的是object_richcompare和long_richcompare

因为A类型中没有定义==方法, 并且在long_richcompare中判断都A类型不是long类型, 所以都返回Py_NotImplemented

所以, 走到比对流程3, 也就是进入直接比对内存地址, 返回False

.. code-block:: python

    In [7]: a > 1
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-7-878db1e46c45> in <module>()
    ----> 1 a > 1
    
    TypeError: '>' not supported between instances of 'A' and 'int'
    
    In [8]: 1 > a
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-8-170f585a84fa> in <module>()
    ----> 1 1 > a
    
    TypeError: '>' not supported between instances of 'int' and 'A'

上面两个比对都报异常, 是因为进入了比对流程3, 在3中直接报异常

.. code-block:: python

    In [9]: b == 1
    Out[9]: False
    
    In [10]: b == 10
    Out[10]: False
    
    In [11]: b > 9
    Out[11]: True


上面的例子中, B类型定义了gt方法, 那么==的话, 依然是走流程2, 3, 流程2中进入的函数是slot_tp_richcompare

因为类型B对应了gt方法, 所以不是object_richcompare, 但是由于没有定义eq方法, 所以还会返回Py_NotImplemented

所以走到了3, 进入了==和!=的比较

而大于(>)操作则是进入到了slot_tp_richcompare, 然后调用了B类型的gt方法, 所以返回True. 这是在流程2中直接返回的

.. code-block:: python

    In [12]: 9 < b
    Out[12]: True

这个例子是进入到流程2中, 发现long类型的对比long_richcompare返回Py_NotImplemented, 所以会调用到

B类型的gt, 这是因为传入的是lt, 所以右参数是gt方法, 也就是调用B类型的\_\_gt\_\_方法

in和==的区别
==============

我们再来看看之前的问题:

为什么 *x in (x,)* 要快于 *x == x* ?

来看看in的时候操作少在哪里?


我们知道, in, not in也是走cmp_outcome函数

.. code-block:: c

    static PyObject *
    cmp_outcome(int op, PyObject *v, PyObject *w)
    {
    
    
        int res = 0;
        switch (op) {
        case PyCmp_IS:
            res = (v == w);
            break;
        case PyCmp_IS_NOT:
            res = (v != w);
            break;
        case PyCmp_IN:
            res = PySequence_Contains(w, v);
            if (res < 0)
                return NULL;
            break;
    
        }
    
    
    }


所以就是函数PySequence_Contains, 而pysequence_contains调用路径是

pysequence_contains -> _PySequence_IterSearch

而在_PySequence_IterSearch中, 拿到seq对象的迭代器对象(iter), 然后调用PyObject_RichCompareBool


.. code-block:: c

    Py_ssize_t
    _PySequence_IterSearch(PyObject *seq, PyObject *obj, int operation)
    {
    
        it = PyObject_GetIter(seq);
        
        for (;;) {
        
            PyObject *item = PyIter_Next(it);
        
            cmp = PyObject_RichCompareBool(obj, item, Py_EQ);
        
        }
    
    }

关键就在于PyObject_RichCompareBool

.. code-block:: c

    int
    PyObject_RichCompareBool(PyObject *v, PyObject *w, int op)
    {
        PyObject *res;
        int ok;
    
        /* Quick result when objects are the same.
           Guarantees that identity implies equality. */
        if (v == w) {
            if (op == Py_EQ)
                return 1;
            else if (op == Py_NE)
                return 0;
        }
    
        res = PyObject_RichCompare(v, w, op);
        if (res == NULL)
            return -1;
        if (PyBool_Check(res))
            ok = (res == Py_True);
        else
            ok = PyObject_IsTrue(res);
        Py_DECREF(res);
        return ok;
    }

可以看到, 如果遍历的元素和传入的元素地址一样, 直接返回真假, 否则调用PyObject_RichCompare

可以看到, PyObject_RichCompare不仅仅要处理==和!=, 还要处理>, >=等等操作, 所以要处理的东西很多

而PyObject_RichCompareBool则是如果两者地址相等, 直接返回, 少了很多判断(相对于PyObject_RichCompare)

所以, 针对stack overflow中的回答:

*Both dispatch to if (left_pointer == right_pointer); the difference is just how much work they do to get there. in just does less.*

进行一点扩充就是:

**所以, 当 *x in (x,)* 的时候, 直接比较地址, 发现相等, 直接返回True, 而 *x == x* 则需要调用对象的魔术方法, 最后才进行内存地址比对**

