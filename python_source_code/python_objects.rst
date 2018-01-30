python对象
============

https://docs.python.org/3/c-api/structures.html

http://www.wklken.me/posts/2014/08/05/python-source-object.html


类
======

类中python的c代码中是用结构体来表示的, 一个结构体中定义了类的属性, 比如dict这个类:

.. code-block:: c

    typedef struct {
        PyObject_HEAD
    
        Py_ssize_t ma_used;
    
        uint64_t ma_version_tag;
    
        PyDictKeysObject *ma_keys;
    
        PyObject **ma_values;
    } PyDictObject;

这个结构就表示dict这个类包含的属性, 也就是为这个结构体分配内存, 就是实例化了dict对象.

下面所说的对象和结构体基本等价.


PyObject_HEAD
==================

其中每一个对象都包含一个PyObject_HEAD结构(比如上面的dictobject), 所以每一个对象都分为两个部分:

1. 一个是PyObject_HEAD, 这是一个公共的头.

2. 一个是自己特有的属性, 在dictobject中, ma_used, ma_keys这些都是自己的属性.


.. code-block:: c

   // cpython/Include/Object.h
   #define PyObject_HEAD  PyObject ob_base;

这个结构只包含一个类型为PyObject, 名字为ob_base的字段, 也就是说, 每一个对象都是基础自PyObject, PyObject是一个最基础的类.


PyObject
============

PyObject是一个基本结构, 任何一个python对象都是继承于PyObject, 结构:

.. code-block:: c

    typedef struct _object {
        _PyObject_HEAD_EXTRA
        Py_ssize_t ob_refcnt;
        struct _typeobject *ob_type;
    } PyObject;


_PyObject_HEAD_EXTRA这个头为:

.. code-block:: c

    #define _PyObject_HEAD_EXTRA 
        struct _object *_ob_next;
        struct _object *_ob_prev;
 
这个头包含的是一个对象链表信息, 和gc有关.

所以PyObject中各个属性为:

1. _PyObject_HEAD_EXTRA是表示gc的对象链表

2. ob_refcnt表示对象的引用计数

3. ob_type是指明该PyObject的类型信息.



对象和类型
=============

python中每一个对象都基础自PyObject, 但是每一个对象的类型又是不同的, 区别类型就是PyObject中的ob_type参数, 这个参数是

_typeobject类型的(下面省略了很多代码):

.. code-block:: c

    typedef struct _typeobject {
        // 类型的名称
        const char *tp_name; 
        // 分配时需要的大小
        Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */
    
        /* Methods to implement standard operations */
    
        // 这些函数看名字就知道了
        destructor tp_dealloc;
        printfunc tp_print;
        getattrfunc tp_getattr;
        setattrfunc tp_setattr;
        PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                        or tp_reserved (Python 3) */
        reprfunc tp_repr;
    
        /* Method suites for standard classes */
    
        PyNumberMethods *tp_as_number;
        PySequenceMethods *tp_as_sequence;
        PyMappingMethods *tp_as_mapping;
    
        /* More standard operations (here for binary compatibility) */
    
        // 这些也是, 看名字就知道了
        hashfunc tp_hash;
        ternaryfunc tp_call;
        reprfunc tp_str;
        getattrofunc tp_getattro;
        setattrofunc tp_setattro;

        // 下面还有很多, 慢慢去看吧
    
    } PyTypeObject;

PyNumberMethods, PySequenceMethods和PyMappingMethods这三个是代表三种类型函数的集合(也是一个结构体了).

比如PySequenceMethods代表了可以作为序列来操作, 其中有一个叫sq_contains的函数, 那么如果有一个类型A, 其中包含了

PySequenceMethods, 并且定义了sq_contains的时候, 那么当我们在py代码调用'a' in A的时候, 就会寻找A中的PySequenceMethods函数集合, 

然后找到sq_contains, 然后调用其定义的函数. 其他两个同理.


**所以, PyTypeObject定义了一个类型的基本信息, 所有的类型, 都是一个PyTypeObject, 然后根据情况赋值其中的字段.**

比如tuple中, 其类型是PyTuple_Type, 下面省略了很多代码, 留下例子:


先来看PyMappingMethods的定义:


.. code-block:: c

    typedef struct {
        lenfunc mp_length;
        binaryfunc mp_subscript;
        objobjargproc mp_ass_subscript; //注意这个位置
    } PyMappingMethods;

然后是PyTuple_Type相关定义(依然省略了很多代码):

.. code-block:: c

    static PyMappingMethods tuple_as_mapping = {
        (lenfunc)tuplelength,
        (binaryfunc)tuplesubscript,
        0
    };
    
    static PyObject *tuple_iter(PyObject *seq);
    
    PyTypeObject PyTuple_Type = {
        // 这个就是tp_name
        "tuple",
        // 这个就是tp_basicsize
        sizeof(PyTupleObject) - sizeof(PyObject *),
        // 这个就是mapping函数集合
        &tuple_as_mapping,                          /* tp_as_mapping */
        // 这个就是调用tuple生成tuple时候调用的函数
        // 其名字是tp_new
        tuple_new,                                  /* tp_new */
    };


PyTuple_Type是一个PyTuple_Type类型, 其中tuple_new这个函数是处于tp_new这个字段的位置, 说明当调用

.. code-block:: python

   t = tuple([1, 2, 3])

的时候, 会调用tp_new, 也就是调用其定义的tuple_new函数.

那么tp_as_mapping位置是定义为tuple_as_mapping, 那么该结构体里面最后一个成员为0, 对比PyMappingMethods结构, 那么就是mp_ass_subscript函数未定义.

那么, 如果我们对上面的元组t赋值, 调用:

.. code-block:: python

   t[0] = 'a'

的时候, 会寻找到PyTuple_Type中的PyMappingMethods中的mp_ass_subscript函数, 然后发现未定义, 那么说明改对象不允许set item, 所以报错.


c代码中强制转换
==================

比如dict中, 会看到类型的代码:

.. code-block:: c

   PyObject *obj;
   PyDictObject *dict;
 
   dict = (PyDictObject *)obj

从上面类的定义可知, 每一个具体类的定义都是带有PyObject的, 并且首个字段就是PyObject, 那么

也就说, 一个PyDictObject的起始地址就是其PyObject的其实地址, 所以两个指针其实指向的是同一个地址, 那么自然可以通过

强制转换得到. 关于C语言指针, 参考: C_指针小结.rst


一些代码结构
=================

执行python代码其实就是执行其对应的字节码的, 字节码具体步骤在cpython/Python/ceval.c中定义了, 比如调用:

.. code-block:: python

    x['a'] = 1

通过dis.dis知道其字节码为STORE_SUBSCR, 在ceval.c中有:

.. code-block:: c

        TARGET(BINARY_SUBSCR) {
            PyObject *sub = POP();
            PyObject *container = TOP();
            PyObject *res = PyObject_GetItem(container, sub);
            Py_DECREF(container);
            Py_DECREF(sub);
            SET_TOP(res);
            if (res == NULL)
                goto error;
            DISPATCH();
        }


所以执行的是PyObject_GetItem这样一个函数, 而往往在字节码中调用的函数是一些通用接口, 比如这个PyObject_GetItem函数

就是说执行对应对象的getitem方法.


这一类统一接口的函数定义在cpython/Objects/abstract.c中, 比如PyObject_GetItem:


.. code-block:: c

    PyObject *
    PyObject_GetItem(PyObject *o, PyObject *key)
    {
        PyMappingMethods *m;
    
        if (o == NULL || key == NULL) {
            return null_error();
        }
    
        m = o->ob_type->tp_as_mapping;
        if (m && m->mp_subscript) {
            PyObject *item = m->mp_subscript(o, key);
            assert((item != NULL) ^ (PyErr_Occurred() != NULL));
            return item;
        }
    
        if (o->ob_type->tp_as_sequence) {
            if (PyIndex_Check(key)) {
                Py_ssize_t key_value;
                key_value = PyNumber_AsSsize_t(key, PyExc_IndexError);
                if (key_value == -1 && PyErr_Occurred())
                    return NULL;
                return PySequence_GetItem(o, key_value);
            }
            else if (o->ob_type->tp_as_sequence->sq_item)
                return type_error("sequence index must "
                                  "be integer, not '%.200s'", key);
        }
    
        return type_error("'%.200s' object is not subscriptable", o);
    }

这个函数首先去检查该对象是包含了tp_as_mapping这个函数集合(也就是这个对象可以当做mapping对象来看待), 调用其mapping函数结构中的mp_subscript方法.

如果该函数没有定义mapping函数集合, 那么转而去寻找对象的tp_as_sequence函数集合(也就是该对象可以当做sequence对象看待), 那么调用其PySequence_GetItem这个接口.

最后通过对象中的type定义的接口, 调用对应的函数


**整个过程都是忽略对象的具体类型, 而是去找对应的接口函数, oop嘛**



PyObject_VAR_HEAD
======================

有些对象包含的是PyObject_VAR_HEAD而不是PyObject_HEAD, 但是这个接头只是PyObject的一些拓展而已.

PyObject_VAR_HEAD的定义是:

.. code-block:: c

    #define PyObject_VAR_HEAD PyVarObject ob_base;

而PyVarObject为:


.. code-block:: c

    typedef struct {
        PyObject ob_base;
        Py_ssize_t ob_size; /* Number of items in variable part */
    } PyVarObject;

所以说, PyVarObject是PyObject和一个表示对象长度的ob_size组合, 也就是说, PyVarObject也是一个PyObject, 只是它表示

该对象有一个长度参数. 根据 `文档 <https://docs.python.org/3/c-api/structures.html#c.PyVarObject>`_ 中的说明, 该头部表示该对象是一个

带有长度的对象. 这个长度会不会变化, 或者这么说, 是否是可变对象, 恩~~看起来都不一定, 因为tuple和list的定义都是包含

PyVarObject, 但是tuple是一个不可变对象, 而list是一个可变对象.

**所以我觉得如果一个对象带有这个头, 只是说明改对象有个长度参数, 初始化的时候会根据该长度初始化空间, 但是也就仅此而已, 跟其他PyObject没说明不同**


