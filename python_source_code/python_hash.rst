####
hash
####

python中调用hash的时候的流程


1. unicode对象的hash是创建的时候是赋值-1, 第一次计算之后就缓存下来了
   
   空unicode的hash是0, 非空使用hash算法计算, 默认hash算法是siphash24

2. long对象的hash就是其值, 计算过程有点绕而已.

3. object对象的hash是其内存地址经过偏移计算后的返回值, 并没有缓存, 每次都计算

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

一般我们都是关闭Py_HASH_CUTOFF配置的, 然后在Ubuntu14.04, python3.6中, 默认的hash算法是siphash24, 可以通过Py_HASH_ALGORITHM宏定义修改.



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

2. 如果long对象大于2**30, 那么得算一下, 过程看注释

3. 如果long对象小鱼2**30, 那么直接返回

4. 之所以是30, 是因为python使用数组来保存无限大数字, 数组每一位最大都是2**30



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
        y = (y >> 4) | (y << (8 * SIZEOF_VOID_P - 4));
        x = (Py_hash_t)y;
        if (x == -1)
            x = -2;
        return x;
    }
    
可以看到, 一般对象的hash就是其内存地址, 进行偏移计算之后的值.

并且没有缓存, 每次都计算的

