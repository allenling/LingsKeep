####
hash
####

python中调用hash的时候的流程

.. [1] https://hynek.me/articles/hashes-and-equality/


1. unicode对象的hash是创建的时候是赋值-1, 第一次计算之后就缓存下来了
   
   空字符串(长度为0)的hash是0, 长度大于0的字符串使用hash算法计算, 默认hash算法是siphash24

2. long对象的hash有, 64位机器下, 基准值x = 2**61, 如果a小于x, 那么hash(a) = a, 如果a>x, a=2**n + m, n > 61,

   有hash(a) = 2**(n-61) + m

3. object对象的hash是其内存地址经过偏移计算后的返回值, 并没有缓存, 每次都计算

   偏移计算: 64位机器下, x是地址p的64整数, 则hash的结果是 x>>4 | x<<60, 结果是负数是因为补码表示法

4. hash只能作用于不可变(immutable)对象, 所以list是不可以被hash的, 而tuple是可以被hash的

   但是, 因为tuple的hash和其中元素的hash有关, 所以, 如果tuple中有可变对象, 比如([1], 2)这样, 那么也不能hash

5. tuple的hash值没有被缓存, 原因看issue #9685, 大概意思就是在经过测试之后, 缓存了tuple的hash值之后
   
   速度并没有明显的提升

6. 定义了\_\_hash\_\_的类, 如果\_\_hash\_\_的返回值变了, 那么有可能in操作不存在

   参考[1]_

内建hash函数
==============

python中的builtin的定义的文件是在cpython/Python/bltinmodule.c

.. code-block:: c

    static PyObject *
    builtin_hash(PyObject *module, PyObject *obj)
    /*[clinic end generated code: output=237668e9d7688db7 input=58c48be822bf9c54]*/
    {
        Py_hash_t x;
    
        x = PyObject_Hash(obj);
        if (x == -1)
            return NULL;
        return PyLong_FromSsize_t(x);
    }

所以主要实现是在PyObject_Hash这个函数上

cpython/Objects/object.c

.. code-block:: c

    Py_hash_t
    PyObject_Hash(PyObject *v)
    {
        PyTypeObject *tp = Py_TYPE(v);
        // ----------看这里
        // 这里直接调用tp_hash函数
        if (tp->tp_hash != NULL)
            return (*tp->tp_hash)(v);
        /* To keep to the general practice that inheriting
         * solely from object in C code should work without
         * an explicit call to PyType_Ready, we implicitly call
         * PyType_Ready here and then check the tp_hash slot again
         */
        if (tp->tp_dict == NULL) {
            if (PyType_Ready(tp) < 0)
                return -1;
            if (tp->tp_hash != NULL)
                return (*tp->tp_hash)(v);
        }
        /* Otherwise, the object can't be hashed */
        return PyObject_HashNotImplemented(v);
    }

所以基本上是hash值是对象中定义的 *tp_hash* 的函数返回值了, 缓存如否依赖于对象的tp_hash的实现


unicode对象
===============

unicode对象的hash是缓存起来的

unicode对象的hash函数是unicode_hash

cpython/Objects/unicodeobject.c


.. code-block:: c

    static Py_hash_t
    unicode_hash(PyObject *self)
    {
        Py_ssize_t len;
        Py_uhash_t x;  /* Unsigned for defined overflow behavior. */
    
    #ifdef Py_DEBUG
        assert(_Py_HashSecret_Initialized);
    #endif
        // ---------------看这里
        if (_PyUnicode_HASH(self) != -1)
            return _PyUnicode_HASH(self);
        if (PyUnicode_READY(self) == -1)
            return -1;
        len = PyUnicode_GET_LENGTH(self);
        /*
          We make the hash of the empty string be 0, rather than using
          (prefix ^ suffix), since this slightly obfuscates the hash secret
        */
        if (len == 0) {
            _PyUnicode_HASH(self) = 0;
            return 0;
        }
        x = _Py_HashBytes(PyUnicode_DATA(self),
                          PyUnicode_GET_LENGTH(self) * PyUnicode_KIND(self));
        // ----------还要看这里----------
        // 这里计算了hash值之后, 赋值给
        // PyUnicodeObject中的hash属性
        _PyUnicode_HASH(self) = x;
        return x;
    }

所以unicode调用的是_PyUnicode_HASH这个宏得到unicode对象的hash属性, 如果是-1, 则说明没有计算过hash, 计算并赋值, 如果hash属性不是-1,

则直接返回.

.. code-block:: c

    #define _PyUnicode_HASH(op)                             \
        (((PyASCIIObject *)(op))->hash)

所以, 这个宏是拿到PyASCIIObject中的hash这个参数, 由于PyUnicodeObject中也包含了PyASCIIObject, 所以可以理解为PyUnicodeObject的hash属性


.. code-block:: c

    typedef struct {
        // 这里包含了PyASCIIObject
        PyASCIIObject _base;
        Py_ssize_t utf8_length;     /* Number of bytes in utf8, excluding the
                                     * terminating \0. */
        char *utf8;                 /* UTF-8 representation (null-terminated) */
        Py_ssize_t wstr_length;     /* Number of code points in wstr, possible
                                     * surrogates count as two code points. */
    } PyCompactUnicodeObject;
    
    // PyUnicodeObject中包含的PyCompactUnicodeObject含有PyASCIIObject结构
    typedef struct {
        PyCompactUnicodeObject _base;
        union {
            void *any;
            Py_UCS1 *latin1;
            Py_UCS2 *ucs2;
            Py_UCS4 *ucs4;
        } data;                     /* Canonical, smallest-form Unicode buffer */
    } PyUnicodeObject;


**而PyASCIIObject对象的hash在一开始的时候是赋值-1, 然后第一次计算之后就保存下来了**


1. 初始化unicode的时候

hash值赋值为-1

.. code-block:: c

    PyObject *
    PyUnicode_New(Py_ssize_t size, Py_UCS4 maxchar)
    {
    
        // 省略了很多代码
        
        _PyUnicode_HASH(unicode) = -1;
        
        // 省略了很多代码
    
    }

2. unicode_hash调用_Py_HashBytes计算unicode的hash

cpython/Python/pyhash.c

.. code-block:: c

    Py_hash_t
    _Py_HashBytes(const void *src, Py_ssize_t len)
    {
        Py_hash_t x;
        /*
          We make the hash of the empty string be 0, rather than using
          (prefix ^ suffix), since this slightly obfuscates the hash secret
        */
        // 这里, 空unicode的hash是固定的0
        if (len == 0) {
            return 0;
        }
    
    #ifdef Py_HASH_STATS
        hashstats[(len <= Py_HASH_STATS_MAX) ? len : 0]++;
    #endif
    
    // hash cutoff的配置
    #if Py_HASH_CUTOFF > 0
        if (len < Py_HASH_CUTOFF) {
            /* Optimize hashing of very small strings with inline DJBX33A. */
            Py_uhash_t hash;
            const unsigned char *p = src;
            hash = 5381; /* DJBX33A starts with 5381 */
    
            switch(len) {
                /* ((hash << 5) + hash) + *p == hash * 33 + *p */
                case 7: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 6: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 5: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 4: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 3: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 2: hash = ((hash << 5) + hash) + *p++; /* fallthrough */
                case 1: hash = ((hash << 5) + hash) + *p++; break;
                default:
                    assert(0);
            }
            hash ^= len;
            hash ^= (Py_uhash_t) _Py_HashSecret.djbx33a.suffix;
            x = (Py_hash_t)hash;
        }
        else
    #endif /* Py_HASH_CUTOFF */
            // 如果没有定义CUTOFF
            x = PyHash_Func.hash(src, len);
    
        if (x == -1)
            return -2;
        return x;
    }

一般我们都是关闭Py_HASH_CUTOFF配置的, 然后在Ubuntu16.04, python3.6中, 默认的hash算法是siphash24, 可以通过Py_HASH_ALGORITHM宏定义修改.



Py_HASH_CUTOFF
================

这个配置是为了对设置的范围长度unicode的优化

cpython/Include/pyhash.h

.. code-block:: c

    /* cutoff for small string DJBX33A optimization in range [1, cutoff).
     *
     * About 50% of the strings in a typical Python application are smaller than
     * 6 to 7 chars. However DJBX33A is vulnerable to hash collision attacks.
     * NEVER use DJBX33A for long strings!
     *
     * A Py_HASH_CUTOFF of 0 disables small string optimization. 32 bit platforms
     * should use a smaller cutoff because it is easier to create colliding
     * strings. A cutoff of 7 on 64bit platforms and 5 on 32bit platforms should
     * provide a decent safety margin.
     */
    #ifndef Py_HASH_CUTOFF
    #  define Py_HASH_CUTOFF 0
    #elif (Py_HASH_CUTOFF > 7 || Py_HASH_CUTOFF < 0)
    #  error Py_HASH_CUTOFF must in range 0...7.
    #endif /* Py_HASH_CUTOFF */

更多请参考 `pep0456 <https://www.python.org/dev/peps/pep-0456/>`_


long对象
==========

long对象的tp_hash函数定义是long_hash

cpython/Objects/longObject.c


.. code-block:: c

    static Py_hash_t
    long_hash(PyLongObject *v)
    {
        Py_uhash_t x;
        Py_ssize_t i;
        int sign;
    
        i = Py_SIZE(v);
        // 这里, 如果long对象的长度(数组)是
        // 0, 则返回0
        // 1, 就直接返回数值
        // -1, 这个没看明白
        switch(i) {
        case -1: return v->ob_digit[0]==1 ? -2 : -(sdigit)v->ob_digit[0];
        case 0: return 0;
        case 1: return v->ob_digit[0];
        }
        sign = 1;
        x = 0;
        if (i < 0) {
            sign = -1;
            i = -(i);
        }
        // 如果i>1, 也就是long对象
        // 至少大于2**30
        // 计算过程看注释吧
        while (--i >= 0) {
            /* Here x is a quantity in the range [0, _PyHASH_MODULUS); we
               want to compute x * 2**PyLong_SHIFT + v->ob_digit[i] modulo
               _PyHASH_MODULUS.
    
               The computation of x * 2**PyLong_SHIFT % _PyHASH_MODULUS
               amounts to a rotation of the bits of x.  To see this, write
    
                 x * 2**PyLong_SHIFT = y * 2**_PyHASH_BITS + z
    
               where y = x >> (_PyHASH_BITS - PyLong_SHIFT) gives the top
               PyLong_SHIFT bits of x (those that are shifted out of the
               original _PyHASH_BITS bits, and z = (x << PyLong_SHIFT) &
               _PyHASH_MODULUS gives the bottom _PyHASH_BITS - PyLong_SHIFT
               bits of x, shifted up.  Then since 2**_PyHASH_BITS is
               congruent to 1 modulo _PyHASH_MODULUS, y*2**_PyHASH_BITS is
               congruent to y modulo _PyHASH_MODULUS.  So
    
                 x * 2**PyLong_SHIFT = y + z (mod _PyHASH_MODULUS).
    
               The right-hand side is just the result of rotating the
               _PyHASH_BITS bits of x left by PyLong_SHIFT places; since
               not all _PyHASH_BITS bits of x are 1s, the same is true
               after rotation, so 0 <= y+z < _PyHASH_MODULUS and y + z is
               the reduction of x*2**PyLong_SHIFT modulo
               _PyHASH_MODULUS. */
            x = ((x << PyLong_SHIFT) & _PyHASH_MODULUS) |
                (x >> (_PyHASH_BITS - PyLong_SHIFT));
            x += v->ob_digit[i];
            if (x >= _PyHASH_MODULUS)
                x -= _PyHASH_MODULUS;
        }
        x = x * sign;
        if (x == (Py_uhash_t)-1)
            x = (Py_uhash_t)-2;
        return (Py_hash_t)x;
    }

所以,

1. 0的hash就是0

2. 看注释计算的过程以及_PyHASH_BITS这个宏的定义在64位平台上是61, 所以就是longobject的hash值, 在2**61之后

   就有点区别了, 看下面的例子


* a > 2**61, 并且a = 2**n, n >= 61, 那么, hash(a) = 2**(n-61)


.. code-block:: python

    In [24]: for i in ['2**31', '2**60', '2**61', '2**62', '2**63', '2**65', '2**90']:
        ...:     int_i = eval(i)
        ...:     if int_i < 2**61:
        ...:         print(i, hash(int_i))
        ...:     else:
        ...:         m = int(i.split('**')[1])
        ...:         left_m = m - 61
        ...:         print(i, hash(int_i), '2**%s = ' % left_m, 2**left_m)
        ...:         
        ...:         
    2**31 2147483648
    2**60 1152921504606846976

    2**61 1                    2**0 =  1
    2**62 2                    2**1 =  2
    2**63 4                    2**2 =  4
    2**65 16                   2**4 =  16
    2**90 536870912            2**29 =  536870912

* 如果a > 61, 并且a = 2**n + m, hash(a) = 2**(n-61) + m

.. code-block:: python

    In [26]: for i in ['2**63', '2**63+1', '2**63+2', '2**63+3']:
        ...:     int_i = eval(i)
        ...:     print(i, hash(int_i))
        ...:     
    2**63   4
    2**63+1 5
    2**63+2 6
    2**63+3 7


object对象
============

**object的hash计算并没有缓存**

任何类都是继承于Object这个类(使用class关键字定义类的时候, 不写父类就是直接隐式继承了), 在c代码中, Object称为基本类型PyBaseObject_Type

cpython/Objects/typeobject.c

.. code-block:: c

    PyTypeObject PyBaseObject_Type = {
        // 省略了代码

        // 这个就是默认的hash方法
        (hashfunc)_Py_HashPointer,                  /* tp_hash */
        // 省略了代码

        // 下面是和gc有关的
        object_init,                                /* tp_init */
        PyType_GenericAlloc,                        /* tp_alloc */
        object_new,                                 /* tp_new */
        PyObject_Del,                               /* tp_free */
    };


所以一般对象, 比如用class定义的, 默认的__hash__方法是_Py_HashPointer:

cpython/Python/pyhash.c

.. code-block:: c

    Py_hash_t
    _Py_HashPointer(void *p)
    {
        Py_hash_t x;
        // 这里把p转成size_t
        // 因为p是指向对象的指针
        // 所以p的值是对象的地址
        // 所以这里就是把对象的地址转成size_t的长度
        size_t y = (size_t)p;
        /* bottom 3 or 4 bits are likely to be 0; rotate y by 4 to avoid
           excessive hash collisions for dicts and sets */
        // 然后下面就是偏移计算的过程了

        // 翻转尾部的四位到头部, 然后或运算
        y = (y >> 4) | (y << (8 * SIZEOF_VOID_P - 4));
        x = (Py_hash_t)y;
        if (x == -1)
            x = -2;
        return x;
    }
    
可以看到, 一般对象的hash就是其内存地址, 进行偏移计算之后的值.

并且没有缓存, 每次都计算的

偏移计算是取尾部的4位, 翻转到头部, 然后取或预算结果

.. code-block:: python

    In [1]: class A:
       ...:     pass
       ...: 
    
    In [2]: a=A()
    
    In [3]: id(a)
    Out[3]: 140103316223088
    
    In [4]: hash(a)
    Out[4]: 8756457263943


1. 其中, 140103316223088 =              0000000000000000011111110110110001011000011001010011010001110000

2. 然后左移去掉后四位,               有 0000000000000000000001111111011011000101100001100101001101000111

3. 然后, 右移60, 把后四位移动到头部, 有 0000000000000000000000000000000000000000000000000000000000000000

4. 然后2, 3或预算, 结果还是2的值,    有 0000000000000000000001111111011011000101100001100101001101000111

5. 最后结果就是: hash(a) = int('0000000000000000000001111111011011000101100001100101001101000111', 2) = 8756457263943

**注意的是**

返回负数是因为, 如果翻转的后四位最高位是1, 比如0000000000000000011111110110110001011000011001010011010001111000

那么需要按补码表示来取值

object的__hash__
--------------------

下面例子来自参考[1]_

.. code-block:: python

    In [43]: c=C()
    
    In [44]: c.data = 1
    
    In [45]: x={}
    
    In [46]: x[c] = 'c'
    
    In [47]: x
    Out[47]: {<__main__.C at 0x7feda0bc8ef0>: 'c'}
    
    In [48]: c.data = 2
    
    In [49]: c in x
    Out[49]: False


x这个dict命名保存了c对象, 然后我们更改c.__hash__的返回值之后, c in x 就是False了


list/tuple
================

list是可变对象, 所以不能hash

.. code-block:: c

    PyTypeObject PyList_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "list",
        sizeof(PyListObject),
        0,
        (destructor)list_dealloc,                   /* tp_dealloc */
        0,                                          /* tp_print */
        0,                                          /* tp_getattr */
        0,                                          /* tp_setattr */
        0,                                          /* tp_reserved */
        (reprfunc)list_repr,                        /* tp_repr */
        0,                                          /* tp_as_number */
        &list_as_sequence,                          /* tp_as_sequence */
        &list_as_mapping,                           /* tp_as_mapping */

        // 看这里!!!!!!!
        PyObject_HashNotImplemented,                /* tp_hash */
    
    
    }

list的tp_hash定义是NotImplemented, 所以hash(list)报错


而tuple是不可变对象, 所以能hash


.. code-block:: c

    PyTypeObject PyTuple_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "tuple",
    
        // 这里
        (hashfunc)tuplehash,                        /* tp_hash */
    
    
    }

tuple的hash函数是tuplehash, 然后注释说不会去缓存tuple的hash值, 在issue #9685中提到了

貌似是因为就算缓存了也没发现性能上有明显的提升

然后tuple的hash值和每一个元素hash值有关, 经过一系列的翻转的出来的


.. code-block:: c

    /* Prime multiplier used in string and various other hashes. */
    #define _PyHASH_MULTIPLIER 1000003UL  /* 0xf4243 */

    /* The addend 82520, was selected from the range(0, 1000000) for
       generating the greatest number of prime multipliers for tuples
       upto length eight:
    
         1082527, 1165049, 1082531, 1165057, 1247581, 1330103, 1082533,
         1330111, 1412633, 1165069, 1247599, 1495177, 1577699
    
       下面一句就是说, 经过测试发现, 就算缓存了性能也灭有明显的提升
       Tests have shown that it's not worth to cache the hash value, see
       issue #9685.
    */
    
    static Py_hash_t
    tuplehash(PyTupleObject *v)
    {
        Py_uhash_t x;  /* Unsigned for defined overflow behavior. */
        Py_hash_t y;
        Py_ssize_t len = Py_SIZE(v);
        PyObject **p;
        Py_uhash_t mult = _PyHASH_MULTIPLIER;
        x = 0x345678UL;
        p = v->ob_item;
        while (--len >= 0) {
            y = PyObject_Hash(*p++);
            if (y == -1)
                return -1;
            x = (x ^ y) * mult;
            /* the cast might truncate len; that doesn't change hash stability */
            mult += (Py_hash_t)(82520UL + len + len);
        }
        x += 97531UL;
        if (x == (Py_uhash_t)-1)
            x = -2;
        return x;
    }




