############
if的判断流程
############


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

