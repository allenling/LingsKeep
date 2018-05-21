##################
Python中的自定义类
##################

类就是类型, 类型就是类!!!!!

自定义类(也就是class关键字), 那么python就会生成一个PyTypeObject对象, 然后实例PyObject的ob_type就是这个PyTypeObject

显然PyTypeObject的ob_type就是PyType_Type

PyType_Type
==============

这个就是python中的type, 也就是每一个对象都基础于该结构

.. code-block:: c

    PyTypeObject PyType_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "type",                                     /* tp_name */
        sizeof(PyHeapTypeObject),                   /* tp_basicsize */
        sizeof(PyMemberDef),                        /* tp_itemsize */
        (destructor)type_dealloc,                   /* tp_dealloc */
        0,                                          /* tp_print */
        0,                                          /* tp_getattr */
        0,                                          /* tp_setattr */
        0,                                          /* tp_reserved */
        (reprfunc)type_repr,                        /* tp_repr */
        0,                                          /* tp_as_number */
        0,                                          /* tp_as_sequence */
        0,                                          /* tp_as_mapping */
        0,                                          /* tp_hash */
        (ternaryfunc)type_call,                     /* tp_call */
        0,                                          /* tp_str */
        (getattrofunc)type_getattro,                /* tp_getattro */
        (setattrofunc)type_setattro,                /* tp_setattro */
        0,                                          /* tp_as_buffer */
        Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
            Py_TPFLAGS_BASETYPE | Py_TPFLAGS_TYPE_SUBCLASS,         /* tp_flags */
        type_doc,                                   /* tp_doc */
        (traverseproc)type_traverse,                /* tp_traverse */
        (inquiry)type_clear,                        /* tp_clear */
        0,                                          /* tp_richcompare */
        offsetof(PyTypeObject, tp_weaklist),        /* tp_weaklistoffset */
        0,                                          /* tp_iter */
        0,                                          /* tp_iternext */
        type_methods,                               /* tp_methods */
        type_members,                               /* tp_members */
        type_getsets,                               /* tp_getset */
        0,                                          /* tp_base */
        0,                                          /* tp_dict */
        0,                                          /* tp_descr_get */
        0,                                          /* tp_descr_set */
        offsetof(PyTypeObject, tp_dict),            /* tp_dictoffset */
        type_init,                                  /* tp_init */
        0,                                          /* tp_alloc */
        type_new,                                   /* tp_new */
        PyObject_GC_Del,                            /* tp_free */
        (inquiry)type_is_gc,                        /* tp_is_gc */
    };

注意下tp_call, tp_new, tp_init


魔术方法
===========

当我们定义了\_\_init\_\_的时候, 会把对应的函数设置到PyType_Type中的属性上

.. code-block:: c

    static slotdef slotdefs[] = {
    
        FLSLOT("__init__", tp_init, slot_tp_init, (wrapperfunc)wrap_init,
               "__init__($self, /, *args, **kwargs)\n--\n\n"
               "Initialize self.  See help(type(self)) for accurate signature.",
               PyWrapperFlag_KEYWORDS),
    
    }


所以, 定义了\_\_init\_\_之后, tp_init就指向了slot_tp_init函数, 如果没有, 那么就是object_init


还有其他魔术方法, 比如\_\_len\_\_:

.. code-block:: c

    static slotdef slotdefs[] = {

    MPSLOT("__len__", mp_length, slot_mp_length, wrap_lenfunc,
           "__len__($self, /)\n--\n\nReturn len(self)."),

    SQSLOT("__len__", sq_length, slot_sq_length, wrap_lenfunc,
           "__len__($self, /)\n--\n\nReturn len(self)."),

    }

如果是mapping类型, 比如dict, 定义了\_\_len\_\_的话, 则\_\_len\_\_就成为as_mapping.mp_length, 指向slot_mp_length, sequence类型也一样


class也是callable
=====================

类也是一个callable对象

.. code-block:: c

    In [94]: A.__call__('data')
    Out[94]: <__main__.A at 0x7fe6ab9626d8>
    
    In [95]: class A:
        ...:     name = 'A_class'
        ...:     def __init__(self, data):
        ...:         self.data = data
        ...:         return
        ...:     
    
    In [96]: callable(A)
    Out[96]: True

    In [97]: A.__call__
    Out[97]: <method-wrapper '__call__' of type object at 0x2777ad8>


然后, 我们生成类实例的时候, 就是调用类的\_\_call\_\_, 也就是tp_call函数



字节码
=========

先看看字节码

.. code-block:: python

    In [11]: dis.dis('''class A:\n    def __init__(self, data):\n        self.data=data\n        return''')
      1           0 LOAD_BUILD_CLASS
                  2 LOAD_CONST               0 (<code object A at 0x7fe6b8bebd20, file "<dis>", line 1>)
                  4 LOAD_CONST               1 ('A')
                  6 MAKE_FUNCTION            0
                  8 LOAD_CONST               1 ('A')
                 10 CALL_FUNCTION            2
                 12 STORE_NAME               0 (A)
                 14 LOAD_CONST               2 (None)
                 16 RETURN_VALUE

dis里面的内容就是:

.. code-block:: python

    class A:
        def __init__(self, data):
            self.data = data
            return

1. LOAD_BUILD_CLASS, 拿到 builtins.\_\_\_build\_class\_\_ 函数, 然后入栈(PUSH)

2. 两个LOAD_CONST分别拿到类的code object和A这个unicode object, 下面的A都是值为'A'的unicode object

3. 生成函数对象, 函数名字就是'A'
   
   **注意这里, 这个函数对象的code object是负责生成一系列方法的, 比如你定义了两个方法, 这个code object**

   **就是编译这两个方法, 然后保存到f->f_locals, 而f->f_locals则是类自己的一个属性字典**

   往下看就比较清楚

4. 8中的LOAD_CONST是再拿到'A', 这里, 拿到的'A'是3中的函数对象
   
   然后我们CALL_FUNCTION, 这里的CALL_FUNCTION是在栈区直接拿的, 而'A'函数则是在CONST, 所以这里调用的是1中入栈的builtins.\_\_\_build\_class\_\_

   然后把'A'函数作为参数传入给builtins.\_\_\_build\_class\_\_

5. 在builtins.\_\_\_build\_class\_\_中, 会先生一个PyTypeObject, 其ob_type=PyType_Type, 然后其tp_name='A', 然后

   执行4中获取的函数, 把PyTypeObject调用\_\_prepare\_\_返回的字典(默认的\_\_prepare\_\_返回空字典)作为globals和locals

   也就是4函数执行的时候, 其f->f_globals和f->f_locals就是类的作用域字典了, 所以, 4中的字节码执行MAKE_FUNCTION去生成各个方法的时候

   保存在f->f_locals, 也就是PyTypeObject自己的属性dict中

6. 经过一系列处理之后, PyTypeObject和PyUnicodeObject('A')就关联起来了!!!!


LOAD_BUILD_CLASS
========================

.. code-block:: c

        TARGET(LOAD_BUILD_CLASS) {
            _Py_IDENTIFIER(__build_class__);

            PyObject *bc;
            if (PyDict_CheckExact(f->f_builtins)) {
                bc = _PyDict_GetItemId(f->f_builtins, &PyId___build_class__);
                if (bc == NULL) {
                    PyErr_SetString(PyExc_NameError,
                                    "__build_class__ not found");
                    goto error;
                }
                Py_INCREF(bc);
            }
            else {
                PyObject *build_class_str = _PyUnicode_FromId(&PyId___build_class__);
                if (build_class_str == NULL)
                    goto error;
                bc = PyObject_GetItem(f->f_builtins, build_class_str);
                if (bc == NULL) {
                    if (PyErr_ExceptionMatches(PyExc_KeyError))
                        PyErr_SetString(PyExc_NameError,
                                        "__build_class__ not found");
                    goto error;
                }
            }
            PUSH(bc);
            DISPATCH();
        }

所以, 就是从builtins里面拿到函数builtin\_\_\_build\_class\_\_, 然后入栈(PUSH)


cpython/Python/bltinmodule.c

.. code-block:: c

    static PyObject *
    builtin___build_class__(PyObject *self, PyObject *args, PyObject *kwds)
    {
        PyObject *func, *name, *bases, *mkw, *meta, *winner, *prep, *ns;
        PyObject *cls = NULL, *cell = NULL;
        Py_ssize_t nargs;

        // 先省略了很多很多代码

        // 这里!!!
        // 这里就是编译函数, 然后绑定到ns, 也就是ns=__prepare__
        cell = PyEval_EvalCodeEx(PyFunction_GET_CODE(func), PyFunction_GET_GLOBALS(func), ns,
                                 NULL, 0, NULL, 0, NULL, 0, NULL,
                                 PyFunction_GET_CLOSURE(func));
        if (cell != NULL) {
            PyObject *margs[3] = {name, bases, ns};
            // 这里!!!!!!!
            // 这里调用PyType_Type->tp_call去生成一个新的PyTypeObject!!
            cls = _PyObject_FastCallDict(meta, margs, 3, mkw);
            if (cls != NULL && PyType_Check(cls) && PyCell_Check(cell)) {
                // 先省略
            }
        }
    error:
        Py_XDECREF(cell);
        Py_DECREF(ns);
        Py_DECREF(meta);
        Py_XDECREF(mkw);
        Py_DECREF(bases);
        return cls;
    }


1. 调用一个函数去编译类中定义的函数, 把类方法都放到ns, 也就是类自己的属性dict中


2. 调用PyType_Type->tp_call -> PyType-Type->tp_new去生成一个新的PyTypeObject

   然后在1的ns, 也就是类自己的属性dict中, 找魔术方法, 比如\_\_init\_\_

   tp_init会被设置为slot_tp_init, 然后slot_tp_init会调用\_\_init\_\_这个python函数

   如果类没有定义\_\_init\_\_, 那么给个默认的tp_init函数, 就是object_init


创建实例
============

我们已经得到一个新的类A, 也就是PyTypeObject('A'), 那么我们生成实例的时候:

.. code-block:: python

    In [15]: dis.dis('''a=A('adata')''')
      1           0 LOAD_NAME                0 (A)
                  2 LOAD_CONST               0 ('adata')
                  4 CALL_FUNCTION            1
                  6 STORE_NAME               1 (a)
                  8 LOAD_CONST               1 (None)
                 10 RETURN_VALUE



会调用A"函数", 因为A(类, PyTypeObejct)也是一个callable对象, 那么会判断都A(PyTypeObject)定义了\_\_call\_\_方法, 所以调用

PyTypeObject->ob_type(PyType_Type)->tp_call

而tp_call会调用tp_new, 调用alloc去分配一个新的内存, 内存中, ob_type就是H这个PyTypeObject, 然后

再调用tp_init, 也就是我们之前说的slot_tp_init, 调动类的\_\_init\_\_这个python函数

.. code-block:: c

    static PyObject *
    type_call(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
    
        // 调用tp_new, 也就是object_new
        obj = type->tp_new(type, args, kwds);
        obj = _Py_CheckFunctionResult((PyObject*)type, obj, NULL);
        if (obj == NULL)
            return NULL;
    
    
        // 调用tp_init, 也就是slot_tp_init
        if (type->tp_init != NULL) {
            int res = type->tp_init(obj, args, kwds);
            if (res < 0) {
                assert(PyErr_Occurred());
                Py_DECREF(obj);
                obj = NULL;
            }
            else {
                assert(!PyErr_Occurred());
            }
        }
    
    }


看看slot_tp_init

.. code-block:: c

    static int
    slot_tp_init(PyObject *self, PyObject *args, PyObject *kwds)
    {
        _Py_IDENTIFIER(__init__);
        // 查找PyType
        PyObject *meth = lookup_method(self, &PyId___init__);
        PyObject *res;
    
        if (meth == NULL)
            return -1;
        res = PyObject_Call(meth, args, kwds);
        Py_DECREF(meth);
        if (res == NULL)
            return -1;
        if (res != Py_None) {
            PyErr_Format(PyExc_TypeError,
                         "__init__() should return None, not '%.200s'",
                         Py_TYPE(res)->tp_name);
            Py_DECREF(res);
            return -1;
        }
        Py_DECREF(res);
        return 0;
    }

lookup_method则会去查找self对象, self已经是一个所谓的实例了, self是一个PyObject

其ob_type=PyTypeObject(tp_name='H'), 然后lookup_method则会根据mro(也就是继承树)去查找

传入的属性名所对应的对象, 这里, \_\_init\_\_在PyTypeObject(tp_name='H')中就找到了

然后调用PyObject_Call去调用\_\_init\_\_对应的py函数

