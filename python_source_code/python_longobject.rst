longobject
=============

python3中, int和long合并了, 都属于longobject

1. 无限长度数字是用数组来实现的

2. 小整数[-5, 255)会被缓存, 所以全局都是同一个对象

3. python3.6不会缓存大整数对象


PyLongObject
==============


cpython/Include/longobject.h

.. code-block:: c

   typedef struct _longobject PyLongObject;

_longobject是在cpython/Include/longintrepr.h

.. code-block:: c

    struct _longobject {
        // 其中head里面就包含了数组长度了.
    	PyObject_VAR_HEAD
        // 又看到了长度为1但是一定会越界的数组
    	digit ob_digit[1];
    };


所以整数就是存储在ob_digit这个数组中的, 每个数组是digit类型, digit是根据PYLONG_BITS_IN_DIGIT宏决定的:

.. code-block:: c

    #if PYLONG_BITS_IN_DIGIT == 30
    typedef uint32_t digit;
    typedef int32_t sdigit; /* signed variant of digit */
    typedef uint64_t twodigits;
    typedef int64_t stwodigits; /* signed variant of twodigits */
    #define PyLong_SHIFT	30
    #define _PyLong_DECIMAL_SHIFT	9 /* max(e such that 10**e fits in a digit) */
    #define _PyLong_DECIMAL_BASE	((digit)1000000000) /* 10 ** DECIMAL_SHIFT */

一般如果没有定义PYLONG_BITS_IN_DIGIT的话, 默认会把PYLONG_BITS_IN_DIGIT设置为30:

.. code-block:: c

    /* If PYLONG_BITS_IN_DIGIT is not defined then we'll use 30-bit digits if all
       the necessary integer types are available, and we're on a 64-bit platform
       (as determined by SIZEOF_VOID_P); otherwise we use 15-bit digits. */
    
    #ifndef PYLONG_BITS_IN_DIGIT
    #if SIZEOF_VOID_P >= 8
    #define PYLONG_BITS_IN_DIGIT 30
    #else
    #define PYLONG_BITS_IN_DIGIT 15
    #endif
    #endif


所以digit类型一般就是4字节的int了, 但是注意的是, 数组的每一个元素只用其30位作为有效位, 也就是每一个数组最大就是2**29.

比如:

1. x=2**30的话, 那么ob_digit长度就是2, 第一个元素是0, 第二个元素就是1, 合起来二进制就是00000(一共30个0)1

2. x=2**70 + 2**40的话, 那么ob_digit长度就是3, 第一个元素是0, 第二个元素是1024(二进制11位, 也就是2**10), 第三个也是1024,

   x展开二进制就是000(30个) 0(19个)1(10个)0  0(19个)1(10个)0


PyLong_FromLong
====================


.. code-block:: c

    PyObject *
    PyLong_FromLong(long ival)
    {
        PyLongObject *v;
        unsigned long abs_ival;
        unsigned long t;  /* unsigned so >> doesn't propagate sign bit */
        int ndigits = 0;
        int sign;
    
        // 小整数就从缓存拿
        CHECK_SMALL_INT(ival);
    
        // 下面是判断符号位的
        if (ival < 0) {
            /* negate: can't write this as abs_ival = -ival since that
               invokes undefined behaviour when ival is LONG_MIN */
            abs_ival = 0U-(unsigned long)ival;
            sign = -1;
        }
        else {
            abs_ival = (unsigned long)ival;
            sign = ival == 0 ? 0 : 1;
        }
    
        /* Fast path for single-digit ints */
        // 构造PyLongObject
        // 这里向右移1位为空表示该整数的ob_digit长度为1, 也就是位数了
        // 也就是大小小于2**30
        // 大于2**30的在下面继续求位数
        if (!(abs_ival >> PyLong_SHIFT)) {
            v = _PyLong_New(1);
            if (v) {
                Py_SIZE(v) = sign;
                v->ob_digit[0] = Py_SAFE_DOWNCAST(
                    abs_ival, unsigned long, digit);
            }
            return (PyObject*)v;
        }
    
    #if PyLong_SHIFT==15
    // 这一部分代码省略了
    #endif
    
        /* Larger numbers: loop to determine number of digits */
        // ob_digit长度超过1的整数继续求位数
        // 向右移动30位不为空, 那么ob_digit长度加1, 也就是位数加1
        t = abs_ival;
        while (t) {
            ++ndigits;
            t >>= PyLong_SHIFT;
        }
        v = _PyLong_New(ndigits);
        if (v != NULL) {
            digit *p = v->ob_digit;
            Py_SIZE(v) = ndigits*sign;
            t = abs_ival;
            while (t) {
                // 每一个ob_digit的元素赋值
                *p++ = Py_SAFE_DOWNCAST(
                    t & PyLong_MASK, unsigned long, digit);
                t >>= PyLong_SHIFT;
            }
        }
        return (PyObject *)v;
    }


小整数池
==========

python中会全局缓存小整数, 缓存的小整数的范围是[-5, 257):


.. code-block:: c

    #ifndef NSMALLPOSINTS
    #define NSMALLPOSINTS           257
    #endif
    #ifndef NSMALLNEGINTS
    #define NSMALLNEGINTS           5
    #endif

    /* Small integers are preallocated in this array so that they
       can be shared.
       The integers that are preallocated are those in the range
       -NSMALLNEGINTS (inclusive) to NSMALLPOSINTS (not inclusive).
    */
    static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];


CHECK_SMALL_INT
----------------

如果是小整数, 则返回

.. code-block:: c

    #define CHECK_SMALL_INT(ival) \
        // 判断大小
        do if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) { \
            return get_small_int((sdigit)ival); \
        } while(0)


get_small_int
------------------

.. code-block:: c

    static PyObject *
    get_small_int(sdigit ival)
    {
        PyObject *v;
        assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
        // 从小整数数组拿出对应数值的对象返回
        v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
        Py_INCREF(v);
    #ifdef COUNT_ALLOCS
        if (ival >= 0)
            quick_int_allocs++;
        else
            quick_neg_int_allocs++;
    #endif
        return v;
    }


py3去掉PyIntBlock
====================

参考1: http://www.wklken.me/posts/2014/08/06/python-source-int.html

参考2: http://www.wklken.me/posts/2014/08/06/python-source-int.html

py2中, dealloc一个整数之后会判断是否是整数, 如果是整数那么回到缓存的free_list, 不是整数则释放内存:

.. code-block:: c

    static void
    int_dealloc(PyIntObject *v)
    {
        if (PyInt_CheckExact(v)) {
            // 这里只要是整数就回到free_list
            Py_TYPE(v) = (struct _typeobject *)free_list;
            free_list = v;
        }
        else
            // 不是整数就释放内存
            Py_TYPE(v)->tp_free((PyObject *)v);
    }


所以py2也是会缓存大整数的, 而py3是直接释放到全局的内存池:

.. code-block:: c

    PyTypeObject PyLong_Type = {
        long_dealloc,                               /* tp_dealloc */
        PyObject_Del,                               /* tp_free */
    };

.. code-block:: c

    static void
    long_dealloc(PyObject *v)
    {
        // 直接调用tp_free, 也就是PyObject_Del
        Py_TYPE(v)->tp_free(v);
    }

PyObject_Del定义为PyObject_Free, 根据python内存中的机制去决定是否去真正释放内存.

具体流程参考: python_memory_management.rst

