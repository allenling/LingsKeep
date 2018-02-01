Tuple
=======

1. 元组创建的时候是根据长度固定分配大小

2. 使用数组来存储元素(地址)

3. 元组不可修改是通过在type中不定义set_item方法来实现的.


----

PyTupleObject
==================

.. code-block:: c

    typedef struct {
        PyObject_VAR_HEAD
        PyObject *ob_item[1];
    } PyTupleObject;

PyObject_VAR_HEAD这个头包含了PyObject和一个长度信息ob_size, PyObject保存了type信息, type指向PyTuple_Type这个结构体.

然后ob_item是一个指针数组, ob_item就是包含了对象的地址了.

ob_item看起来是只是一个长度为1的数组, 但是其实是可以越界的, 关于指针数组已经其越界问题, 参考: `C_指针小结.rst <https://github.com/allenling/LingsKeep/blob/master/C_%E6%8C%87%E9%92%88%E5%B0%8F%E7%BB%93.rst>`_.


初始化
=============

根据长度, 初始化数组然后赋值.


cpython/Objects/tupleobject.c

.. code-block:: c

    PyObject *
    PyTuple_New(Py_ssize_t size)
    {
        PyTupleObject *op;
        Py_ssize_t i;
        if (size < 0) {
            PyErr_BadInternalCall();
            return NULL;
        }
        // 如果size是0, 则拿全局唯一的空tuple
        // 然后返回
        if (size == 0 && free_list[0]) {
            op = free_list[0];
            Py_INCREF(op);
            tuple_zero_allocs++;
            return (PyObject *) op;
        }

        // 从free_list中拿到可用的tuple
        if (size < PyTuple_MAXSAVESIZE && (op = free_list[size]) != NULL) {
            free_list[size] = (PyTupleObject *) op->ob_item[0];
            numfree[size]--;
            // 然后返回
            _Py_NewReference((PyObject *)op);
        }
        else
        {
            // 从内存中, 新分配一个tuple
            op = PyObject_GC_NewVar(PyTupleObject, &PyTuple_Type, size);
            if (op == NULL)
                return NULL;
        }
        // 然后这里根据长度
        // 初始化数组为NULL
        for (i=0; i < size; i++)
            op->ob_item[i] = NULL;
        // gc跟踪, 加入0代链表
        _PyObject_GC_TRACK(op);
        // 返回
        return (PyObject *) op;
    }


赋值空tuple
===============

这里的赋值空tuple是c代码级别赋值PyTupleObject->ob_item数组的

BUILD_TUPLE
-------------

这个字节码是调用:

.. code-block:: python

   a = 1
   b = 2
   x = (a, b)

的时候生成tuple对象的, 字节码操作流程为:

cpython/Python/ceval.c


.. code-block:: c

        TARGET(BUILD_TUPLE) {
            PyObject *tup = PyTuple_New(oparg);
            if (tup == NULL)
                goto error;
            while (--oparg >= 0) {
                PyObject *item = POP();
                PyTuple_SET_ITEM(tup, oparg, item);
            }
            PUSH(tup);
            DISPATCH();
        }

这里直接调用PyTuple_SET_ITEM这个宏来赋值ob_item:


.. code-block:: 

    #define PyTuple_SET_ITEM(op, i, v) (((PyTupleObject *)(op))->ob_item[i] = v)

列表初始化tuple
-----------------

当我们调用tuple(list):

.. code-block:: python

   x = [1, 2, 3]
   t = tuple(x)


这个时候是调用到PyTuple_Type.tp_new, 也就是tuple_new:


.. code-block:: c

    static PyObject *
    tuple_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
        PyObject *arg = NULL;
        // 如果序列为空, 返回一个全局唯一空tuple
        if (arg == NULL)
            return PyTuple_New(0);
        else
            # 这里根据序列返回tuple
            return PySequence_Tuple(arg);
    }


PySequence_Tuple最终是调用PyTuple_New来初始化一个tuple, 然后把序列中的元素地址赋值到ob_item中:


.. code-block:: c

    PyObject *
    PyList_AsTuple(PyObject *v)
    {
        PyObject *w;
        PyObject **p, **q;
        Py_ssize_t n;
        n = Py_SIZE(v);
        w = PyTuple_New(n);
        if (w == NULL)
            return NULL;
        // 下面两个数组赋值
        p = ((PyTupleObject *)w)->ob_item;
        q = ((PyListObject *)v)->ob_item;
        while (--n >= 0) {
            Py_INCREF(*q);
            *p = *q;
            p++;
            q++;
        }
        return w;
    }

赋值就把列表对应对象的地址赋值到tuple对应的数组上.

PyObject_GC_NewVar只是为PyTupleObject分配足够的内存, 然后PyTupleObject的type指向PyTuple_Type,

赋值PyTupleObject的长度size, 为ob_item这个数组分配一个地址.


修改tuple
===================


如果我们修改tuple:

.. code-block:: python

   x= (1, 2, 3)
   x[0] = 'a'
   x[1] += 1

不管是x[0]='a'还是x[1]+=1, 字节码都是STORE_SUBSCR


STORE_SUBSCR
---------------

cpython/Python/ceval.c

.. code-block:: c

        TARGET(STORE_SUBSCR) {
            PyObject *sub = TOP();
            PyObject *container = SECOND();
            PyObject *v = THIRD();
            int err;
            STACKADJ(-3);
            /* container[sub] = v */
            // 最重要的是调用这个函数
            err = PyObject_SetItem(container, sub, v);
            Py_DECREF(v);
            Py_DECREF(container);
            Py_DECREF(sub);
            if (err != 0)
                goto error;
            DISPATCH();
        }

最终会调用到PyObject_SetItem这个函数


PyObject_SetItem
--------------------

这个函数是一个标准的接口, 这个接口的作用是调用type对应的各种set_item方法.

.. code-block:: c

    int
    PyObject_SetItem(PyObject *o, PyObject *key, PyObject *value)
    {
        PyMappingMethods *m;
        // mapping对象的赋值
        m = o->ob_type->tp_as_mapping;
        if (m && m->mp_ass_subscript)
            return m->mp_ass_subscript(o, key, value);
        // 序列对象的赋值
        if (o->ob_type->tp_as_sequence) {
            if (PyIndex_Check(key)) {
                Py_ssize_t key_value;
                key_value = PyNumber_AsSsize_t(key, PyExc_IndexError);
                if (key_value == -1 && PyErr_Occurred())
                    return -1;
                // 如果对象有序列对应的方法, 调用一下
                return PySequence_SetItem(o, key_value, value);
            }
            else if (o->ob_type->tp_as_sequence->sq_ass_item) {
                type_error("sequence index must be "
                           "integer, not '%.200s'", key);
                return -1;
            }
        }
    
        type_error("'%.200s' object does not support item assignment", o);
        return -1;
    }
 
由于tuple也是一个sequence对象, 自然定义了tp_as_sequence, 调用PySequence_SetItem


PySequence_SetItem
-------------------

这个函数会调用序列类对象的序列方法中的seq_ass_item来赋值:

cpython/Objects/abstract.c

.. code-block:: c

    int
    PySequence_SetItem(PyObject *s, Py_ssize_t i, PyObject *o)
    {
        PySequenceMethods *m;
    
        // 找一下sq_ass_item!!!!
        m = s->ob_type->tp_as_sequence;
        if (m && m->sq_ass_item) {
            if (i < 0) {
                if (m->sq_length) {
                    Py_ssize_t l = (*m->sq_length)(s);
                    if (l < 0)
                        return -1;
                    i += l;
                }
            }
            return m->sq_ass_item(s, i, o);
        }
    
        // 没有就报错!!!
        type_error("'%.200s' object does not support item assignment", s);
        return -1;
    }


但是, PyTuple_Type的sequence方法没有定义set_item:

.. code-block:: c

    static PySequenceMethods tuple_as_sequence = {
        // 这里, sq_ass_item没有
        0,                                          /* sq_ass_item */
        0,                                          /* sq_ass_slice */
        (objobjproc)tuplecontains,                  /* sq_contains */
    };

所以tuple的赋值, 在python代码中会报错的!!.


