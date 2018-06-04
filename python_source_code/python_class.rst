##################
Python中的自定义类
##################

类就是类型, 类型就是类!!!!!

自定义类(也就是class关键字), 那么python就会生成一个PyTypeObject对象, 然后实例PyObject的ob_type就是这个PyTypeObject

显然PyTypeObject的ob_type就是PyType_Type

参考:

.. [1] http://hyry.dip.jp/tech/slice/slice.html/33


参考1是类中获取属性的时候, 使用全局数组缓存了最近使用的方法, 其中一些数据变了(因为文中是python2.版本的)

思路没有变

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

   保存在f->f_locals, 也就是局部作用域.

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


对象属性查询
=================

搜索对象属性的时候, 先搜索出类是否有该属性, 然后再搜索对象的__dict__是否有该属性, 然后优先去对象中__dict__的属性值

字节码

.. code-block:: python

    In [35]: dis.dis("a.data")
      1           0 LOAD_NAME                0 (a)
                  2 LOAD_ATTR                1 (data)
                  4 RETURN_VALUE


LOAD_ATTR是调用PyObject_GetAttr

.. code-block:: c

        TARGET(LOAD_ATTR) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *owner = TOP();
            PyObject *res = PyObject_GetAttr(owner, name);
            Py_DECREF(owner);
            SET_TOP(res);
            if (res == NULL)
                goto error;
            DISPATCH();
        }


而PyObject_GetAttr中, 首先去找tp_getattro, 然后是tp_getattr, 否则报错, 一般性对象, 注意, 这里是对象, 都有tp_getattro

其中tp_geattro指向PyObject_GenericGetAttr

.. code-block:: c

    PyObject *
    PyObject_GetAttr(PyObject *v, PyObject *name)
    {
        PyTypeObject *tp = Py_TYPE(v);
    
        if (!PyUnicode_Check(name)) {
            PyErr_Format(PyExc_TypeError,
                         "attribute name must be string, not '%.200s'",
                         name->ob_type->tp_name);
            return NULL;
        }

        // 这里调用tp_getattro函数!!!!!!!!!!!!!!!
        if (tp->tp_getattro != NULL)
            return (*tp->tp_getattro)(v, name);
        if (tp->tp_getattr != NULL) {
            char *name_str = PyUnicode_AsUTF8(name);
            if (name_str == NULL)
                return NULL;
            return (*tp->tp_getattr)(v, name_str);
        }
        PyErr_Format(PyExc_AttributeError,
                     "'%.50s' object has no attribute '%U'",
                     tp->tp_name, name);
        return NULL;
    }

而一般, PyObject->ob_type->tp_getattro则是PyObject_GenericGetAttr, 调用路径是

PyObject_GenericGetAttr -> _PyObject_GenericGetAttrWithDict -> _PyType_Lookup

而_PyObject_GenericGetAttrWithDict中, 先调用_PyType_Lookup去查找类的属性, 再查找对象的__dict__是否有对应的属性值

**_PyType_Lookup中是具体查找类属性, 会遍历mro的过程, 但是有缓存的, 注意, 这里是类属性查找**

然后优先返回对象的__dict__的属性值

.. code-block:: c

    PyObject *
    _PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict)
    {
    
        // _PyType_Lookup去查找类属性
        descr = _PyType_Lookup(tp, name);
        
        if (descr != NULL) {
            // 如果存在, 则校验一下, 省略校验是否是property

            // 如果校验通过, 直接走到done, 也就是返回值
        }
        
        // 然后继续查找对象__dict__
        if (dict == NULL) {
            /* Inline _PyObject_GetDictPtr */
            dictoffset = tp->tp_dictoffset;
        
            // 省略代码
        
            // 所以, 对象的__dict__也不是存储在对象中
            // 而是和unicode object一样, 存储在隔壁
            // 因为这里是object的地址, 往后移动一个距离, 得到dict, 也就是__dict__
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        
        }
        
        // 如果__dict__不为空, 那么查找__dict__
        if (dict != NULL) {
            Py_INCREF(dict);
            res = PyDict_GetItem(dict, name);
            // 如果__dict__中存在属性, 则返回属性
            if (res != NULL) {
                Py_INCREF(res);
                Py_DECREF(dict);
                goto done;
            }
            Py_DECREF(dict);
        }
        
        // 省略代码
        
        // 如果类中有属性, 显然__dict__中不存在属性
        if (descr != NULL) {
            // 那么res=类属性
            // 最后会返回res, 也就是类属性
            res = descr;
            descr = NULL;
            goto done;
        }
    
    }


类属性查询
============

类属性查询和对象属性查询相同点就是

1. 字节码都是LOAD_ATTR
   
2. 都会调用_PyType_Lookup去查询mro
   
但是有一点区别, 区别就是类的tp_getattro是函数type_getattro, 这是因为类是一个PyTypeObejct, 其ob_type是PyType_Type

而PyType_Type的tp_getattro就是tp_getattro

流程就是:

1. 搜索类属性的时候先去搜索meta_type, 一般是PyType_Type, 一般也找不到, 除非是搜索的是内置的PyType_Type属性, 

2. 然后再次搜索自己, 而搜索自己的时候, 因为_PyType_Lookup会搜索mro, 搜索搜索自己的时候就是搜索自己和继承树

3. 1, 2中的搜索函数都是_PyType_Lookup

.. code-block:: c

    /* This is similar to PyObject_GenericGetAttr(),
       but uses _PyType_Lookup() instead of just looking in type->tp_dict. */
    static PyObject *
    type_getattro(PyTypeObject *type, PyObject *name)
    {
        // 得到metatype, 一般就是PyType_Type
        PyTypeObject *metatype = Py_TYPE(type);

        // 省略代码
    
        // 搜索metatype, 一般都是PyType_Type
        /* Look for the attribute in the metatype */
        meta_attribute = _PyType_Lookup(metatype, name);
    
        if (meta_attribute != NULL) {
            // 如果metatype中存在属性, 校验是否是property
            // 省略了校验过程
            // 如果校验成功, 那么直接返回
        }
    
        // 否则搜索自己
        /* No data descriptor found on metatype. Look in tp_dict of this
         * type and its bases */
        attribute = _PyType_Lookup(type, name);
        if (attribute != NULL) {
            // 如果找到了属性
            // 就可以返回
            /* Implement descriptor functionality, if any */
            descrgetfunc local_get = Py_TYPE(attribute)->tp_descr_get;
    
            Py_XDECREF(meta_attribute);
    
            if (local_get != NULL) {
                /* NULL 2nd argument indicates the descriptor was
                 * found on the target object itself (or a base)  */
                return local_get(attribute, (PyObject *)NULL,
                                 (PyObject *)type);
            }
    
            Py_INCREF(attribute);
            // 返回当前类的属性
            return attribute;
        }

        // 后面还是校验的过程和报错, 先省略

        return NULL;
    }


属性查询缓存
=============

  Python的确需要通过继承树搜索属性，但是它会缓存最近的1024个搜索结果，如果没有下标冲突问题，这样做能极大提高循环中对某几个属性的访问
  
  但是如果存在下标冲突，则访问速度又降回到无缓存的情况，会有一定的速度损失。如果你真的很在乎属性访问速度，那么可以在进行大量循环之前
  
  将所有要访问的属性用局域变量进行缓存，这应该是最快捷的方案了
  
  --- 参考1

在查找类属性的时候, 是函数_PyType_Lookup

.. code-block:: c

    /* Internal API to look for a name through the MRO.
       This returns a borrowed reference, and doesn't set an exception! */
    PyObject *
    _PyType_Lookup(PyTypeObject *type, PyObject *name)
    {
        Py_ssize_t i, n;
        PyObject *mro, *res, *base, *dict;
        unsigned int h;
    
        // 先直接从cache里面找
        if (MCACHE_CACHEABLE_NAME(name) &&
            PyType_HasFeature(type, Py_TPFLAGS_VALID_VERSION_TAG)) {
            // 下面是计算hash, 然后得到下标
            // 查看数组对应下标中的保存的类(也就是tp_version_tag)和属性名是否一致
            /* fast path */
            h = MCACHE_HASH_METHOD(type, name);
            if (method_cache[h].version == type->tp_version_tag &&
                method_cache[h].name == name) {
    #if MCACHE_STATS
                method_cache_hits++;
    #endif
                return method_cache[h].value;
            }
        }

        // 找不到从mro中, 也就是继承树上找
        /* Look in tp_dict of types in MRO */
        mro = type->tp_mro;

        if (mro == NULL) {

        }
    
    }


比较的时候, 会通过属性名的hash, 得到一个缓存数组的下标, 然后看看数组对应下标下的元素是否是

我们需要查找的, 也就是比较tp_version_tag和name

而数组下标的计算是宏MCACHE_HASH_METHOD

.. code-block:: c

    #define MCACHE_MAX_ATTR_SIZE    100
    #define MCACHE_SIZE_EXP         12
    #define MCACHE_HASH(version, name_hash)                                 \
            (((unsigned int)(version) ^ (unsigned int)(name_hash))          \
             & ((1 << MCACHE_SIZE_EXP) - 1))
    
    #define MCACHE_HASH_METHOD(type, name)                                  \
            MCACHE_HASH((type)->tp_version_tag,                     \
                        ((PyASCIIObject *)(name))->hash)
    #define MCACHE_CACHEABLE_NAME(name)                             \
            PyUnicode_CheckExact(name) &&                           \
            PyUnicode_IS_READY(name) &&                             \
            PyUnicode_GET_LENGTH(name) <= MCACHE_MAX_ATTR_SIZE


所以, python会缓存最近访问的4096(1<<12 = 2**12 = 4096)个属性, 然后属性名长度不能大于100

其中MCACHE_HASH是计算缓存数组下标的, 计算方式就是(version ^ name_hash) & (1<<12 - 1)

也就是version ^ name_hash的值, 假设是x, 然后x & (1<<12 - 1) = x & (4096 - 1) = x & 4095

我们把4095在64位下展开, 就是000...000(52个0)111...111(12个1), 所以就是取x的低12位

上面的一些参数和参考[1]_中有区别, 参考1中只存1024个属性, 然后计算方式是x的值往右移 8*4 - 10 == 22(可以看成取高10位?)

最后, 当从mro中找到属性之后, 把属性名和tp_version_tag放入到cache数组中

.. code-block:: c

    PyObject *
    _PyType_Lookup(PyTypeObject *type, PyObject *name)
    {
        mro = type->tp_mro;

        //省略mro的搜索过程
    
        if (MCACHE_CACHEABLE_NAME(name) && assign_version_tag(type)) {
            // 计算下标
            h = MCACHE_HASH_METHOD(type, name);
            // 保存到cache数组
            method_cache[h].version = type->tp_version_tag;
            method_cache[h].value = res;  /* borrowed */
            Py_INCREF(name);
            assert(((PyASCIIObject *)(name))->hash != -1);
        #if MCACHE_STATS
                if (method_cache[h].name != Py_None && method_cache[h].name != name)
                    method_cache_collisions++;
                else
                    method_cache_misses++;
        #endif
                Py_SETREF(method_cache[h].name, name);
        }
    
    }

property
============

我们看到, 在搜索属性的时候, 总是会校验对象的类属性, 或者类的metatype所返回的属性, 或者得到类/对象本身的属性值的时候

总是有一个校验过程是, 而这个校验过程是和property有关的

https://stackoverflow.com/questions/17330160/how-does-the-property-decorator-work

https://docs.python.org/3/howto/descriptor.html

property返回一个property对象, property对象是属于data descriptor的数据模型, data descriptor是这样一个定义了__get__, __set__, __delete__方法的对象

.. code-block:: python

    In [1]: import inspect
    
    In [2]: def my_getter(self): return 'in my_getter'
    
    In [3]: prop = property(my_getter)
    
    In [4]: type(prop)
    Out[4]: property
    
    In [5]: inspect.isdatadescriptor(prop)
    Out[5]: True


data descriptor的特点呢, 就是如果一个 **某个类的属性是data descriptor**, 那么当访问 **这个类** 该属性的时候, 会调用data descriptor的__get__方法, setter, deleter同理.


.. code-block:: python

    In [13]: class C:
        ...:     x = prop
        ...:     
    
    In [14]: C.x
    Out[14]: <property at 0x7fec39a68318>
    
    In [15]: c=C()
    
    In [16]: c.x
    Out[16]: 'in my_getter'


可以看到, 定义C类中的一个属性x为property(data descriptor), 然后如果用C.x访问，就是一个property对象, 如果是实例c访问, c.x, 那么会直接调用prop这个property(data descriptor)

的__get__方法, 会调用property对象定义的getter方法, 也就是my_getter函数


要注意的是, **property(data descriptor)只有定义在类中才有用!!**


.. code-block:: python

    In [17]: def my_getter(self): return 'in my_getter'
    
    In [18]: prop = property(my_getter)
    
    In [19]: class C:
        ...:     def __init__(self):
        ...:         self.x = prop
        ...:         
    
    In [20]: C.x
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-20-64cce3573e3e> in <module>()
    ----> 1 C.x
    
    AttributeError: type object 'C' has no attribute 'x'
    
    In [21]: c=C()
    
    In [22]: c.x
    Out[22]: <property at 0x7fec3872b408>


上面的例子中, x这个property是在__init__中赋值给实例属性的, 而不是类属性, 所以直接c.x的时候, 得到的是property对象, 而不是期望的in my_getter字符串.

**property只有在类中才能生效, 推测这是因为property的各个魔术方法(__get__, __set__, __delete__)都接收一个instance和owner, owner就是类, 也就是说property是跟类绑定的**

.. code-block:: python

    In [31]: prop.__get__?
    Signature:      prop.__get__(instance, owner, /)
    Call signature: prop.__get__(*args, **kwargs)
    Type:           method-wrapper
    String form:    <method-wrapper '__get__' of property object at 0x7fec3872b408>
    Docstring:      Return an attribute of instance, which is of type owner.
    
    In [32]: prop.__get__(c, C)
    Out[32]: 'in my_getter'

__get__方法接收owner, 也就是一个类, 比如我们直接调用prop.__get__(c, C)就直接调用了定义的getter方法


实例中动态设置类property
---------------------------

可以动态地在实例中设置类的property, 使用self.__class__就可以了

.. code-block:: python

    In [23]: class C:
        ...:     def __init__(self):
        ...:         c_class = self.__class__
        ...:         c_class.x = prop
        ...:         return
        ...:     
    
    In [24]: C.x
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-24-64cce3573e3e> in <module>()
    ----> 1 C.x
    
    AttributeError: type object 'C' has no attribute 'x'
    
    In [25]: c=C()
    
    In [26]: c.x
    Out[26]: 'in my_getter'


所以, 总结起来就是, property只有类属性的时候才会调用property(data descriptor)对象的__get__方法, 如果某个实例属性被定义为property, 那也是无效的, 访问该属性只能得到property对象

可以在实例中动态地使用self.__class__来将property赋值给类


最后, 我们回过去看看获取类属性的流程, 以它为例子看看property的校验

.. code-block:: c


    static PyObject *
    type_getattro(PyTypeObject *type, PyObject *name)
    {
    
        attribute = _PyType_Lookup(type, name);
        if (attribute != NULL) {
            /* Implement descriptor functionality, if any */
            // 这里, 如果拿到的属性是data descriptor, 那么拿到descriptor
            // 中的tp_descr_get方法
            descrgetfunc local_get = Py_TYPE(attribute)->tp_descr_get;
    
            Py_XDECREF(meta_attribute);
    
            // 如果存在tp_desc_get存在, 直接调用然后返回
            if (local_get != NULL) {
                /* NULL 2nd argument indicates the descriptor was
                 * found on the target object itself (or a base)  */
                return local_get(attribute, (PyObject *)NULL,
                                 (PyObject *)type);
            }
    
            Py_INCREF(attribute);
            return attribute;
        }
    
    }



