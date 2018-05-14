####################
python中函数
####################


创建函数的过程
=====================

一般, 当你在shell中, 敲下回车(定义函数和对象等操作是两次回车)的时候, 就会触发python的语法解析, 把语句解析成code object, 然后执行code object

定义函数也不例外


.. code-block:: python

    In [12]: n = '''def test():
        ...:     a = 1
        ...:     return a
        ...: '''
    
    In [13]: dis.dis(n)
      1           0 LOAD_CONST               0 (<code object test at 0x7f06b8047f60, file "<dis>", line 1>)
                  2 LOAD_CONST               1 ('test')
                  4 MAKE_FUNCTION            0
                  6 STORE_NAME               0 (test)
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE


**其中, 第一个LOAD_CONST就是我们定义的test函数的code object吗?, 如果是的话, 那么函数的code object的编译过程怎么没有呢?**

在README.rst中, 提到在shell模式下, 你每输入一行, python就会解析该行:

比如你要定义函数, 第一行输入 def test():, 然后回车, 那么python从PyOS_Readline中, 就得到这个语句, 然后发现是定义函数, 则

继续, 然后你输入第二行是4个空格+一个语句, 比如    a = 1, 那么python再次解析这行, 然后发现当前是函数定义, 然后继续

所以就是, 在函数定义的时候, 你每输入一行, python就把语句(a=1)编译成字节码, 加入到函数的code object中的字节码中, code_object->co_code,

直到你定义完函数(输入两个回车), 然后此时被编译成另外一组字节码, 就是我们在上面看到的dis.dis的输出

也就是, 函数定义会生成一个code object, 然后存储起来, 函数定义完成之后, 又生成一组字节码, 是关联函数code object和函数名称的

所以

1. 第一个LOAD_CONST则python解析函数定义的每一行之后, 生成的函数的code object, 也就是例子中, 这个code object就是test函数的code object

2. 然后LOAD_CONST, 加载字符串对象test

3. 然后MAKE_FUNCTION, 生成function对象

4. 然后把函数和函数名给对应起来, STORE_NAME


所以, 我们来看看MAKE_FUNCTION


MAKE_FUNCTION
==================

生成function object的地方, 先看看字节码的执行流程

.. code-block:: c

        TARGET(MAKE_FUNCTION) {
            // 显然, qualname和codeobj就是
            // 函数的名字和code object对象了
            PyObject *qualname = POP();
            PyObject *codeobj = POP();
            // 重点是PyFunction_NewWithQualName函数!!!!!!!!!!!!!!!
            PyFunctionObject *func = (PyFunctionObject *)
                PyFunction_NewWithQualName(codeobj, f->f_globals, qualname);

            Py_DECREF(codeobj);
            Py_DECREF(qualname);
            if (func == NULL) {
                goto error;
            }

            // 后续是绑定func对象属性的地方
            if (oparg & 0x08) {
                assert(PyTuple_CheckExact(TOP()));
                func ->func_closure = POP();
            }
            if (oparg & 0x04) {
                assert(PyDict_CheckExact(TOP()));
                func->func_annotations = POP();
            }
            if (oparg & 0x02) {
                assert(PyDict_CheckExact(TOP()));
                func->func_kwdefaults = POP();
            }
            if (oparg & 0x01) {
                assert(PyTuple_CheckExact(TOP()));
                func->func_defaults = POP();
            }

            PUSH((PyObject *)func);
            DISPATCH();
        }

所以, 生成function对象是PyFunction_NewWithQualName函数


PyFunction_NewWithQualName
================================

cpython/Objects/funcobject.c

.. code-block:: c

    PyObject *
    PyFunction_NewWithQualName(PyObject *code, PyObject *globals, PyObject *qualname)
    {
        PyFunctionObject *op;
        PyObject *doc, *consts, *module;
        static PyObject *__name__ = NULL;
    
        if (__name__ == NULL) {
            __name__ = PyUnicode_InternFromString("__name__");
            if (__name__ == NULL)
                return NULL;
        }
    
        // 先分配一个PyFunctionObject内存
        op = PyObject_GC_New(PyFunctionObject, &PyFunction_Type);
        if (op == NULL)
            return NULL;
    
        // 分别设置function对象的各个属性
        op->func_weakreflist = NULL;

        // 绑定function的字节码
        // 也就是code object
        Py_INCREF(code);
        op->func_code = code;

        // 绑定global
        Py_INCREF(globals);
        op->func_globals = globals;

        // 从code object中拿到co_name
        // 赋值为函数的函数名func_name
        op->func_name = ((PyCodeObject *)code)->co_name;
        Py_INCREF(op->func_name);

        // 其他属性
        // 注意的是, 一下几个属性是在MAKE_FUNCTION流程上赋值的
        // 不是在这里赋值, 所以这里都赋值为NULL
        op->func_defaults = NULL; /* No default arguments */
        op->func_kwdefaults = NULL; /* No keyword only defaults */
        op->func_closure = NULL;
    
        // 保存co_consts
        consts = ((PyCodeObject *)code)->co_consts;
        if (PyTuple_Size(consts) >= 1) {
            doc = PyTuple_GetItem(consts, 0);
            if (!PyUnicode_Check(doc))
                doc = Py_None;
        }
        else
            doc = Py_None;
        Py_INCREF(doc);
        op->func_doc = doc;
    
        op->func_dict = NULL;
        op->func_module = NULL;
        op->func_annotations = NULL;
    
        /* __module__: If module name is in globals, use it.
           Otherwise, use None. */

        // 设置function的__module__属性
        module = PyDict_GetItem(globals, __name__);
        if (module) {
            Py_INCREF(module);
            op->func_module = module;
        }
        if (qualname)
            op->func_qualname = qualname;
        else
            op->func_qualname = op->func_name;
        Py_INCREF(op->func_qualname);
    
        _PyObject_GC_TRACK(op);
        return (PyObject *)op;
    }


最后关联函数名字和function obejct
=====================================

字节码是STORE_NAME

.. code-block:: c

        TARGET(STORE_NAME) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *v = POP();
            PyObject *ns = f->f_locals;
            int err;
            if (ns == NULL) {
                PyErr_Format(PyExc_SystemError,
                             "no locals found when storing %R", name);
                Py_DECREF(v);
                goto error;
            }
            if (PyDict_CheckExact(ns))
                err = PyDict_SetItem(ns, name, v);
            else
                err = PyObject_SetItem(ns, name, v);
            Py_DECREF(v);
            if (err != 0)
                goto error;
            DISPATCH();
        }


1. 拿到name, 也就是函数名, 例子中的字符串对象test

2. POP, 拿到之前生成的function object

3. 拿到frame的f_locals, f->locals, 显然是一个字典

4. 把函数和名字存储到f->locals中


关于加载函数, 去看看python_frame_codeobject_builtin.rst


\_\_defaults\_\_
=====================

**函数的默认值既存在code object中(co_consts中), 也存储在function object中**

先来看看默认值函数定义时候的字节码:

.. code-block:: python

    In [13]: nm = '''def default_func(a, b=1):
        ...:     return a, b
        ...: '''
    
    In [14]: dis.dis(nm)
      1           0 LOAD_CONST               4 ((1,))
                  2 LOAD_CONST               1 (<code object default_func at 0x7f45ea7f4c90, file "<dis>", line 1>)
                  4 LOAD_CONST               2 ('default_func')
                  6 MAKE_FUNCTION            1
                  8 STORE_NAME               0 (default_func)
                 10 LOAD_CONST               3 (None)
                 12 RETURN_VALUE


可以看到, 先生成一个consts, 是一个tuple结构, 然后在MAKE_FUNCTION字节码流程可以看到, 

.. code-block:: c

            if (oparg & 0x01) {
                assert(PyTuple_CheckExact(TOP()));
                func->func_defaults = POP();
            }

因为在code object中已经把默认值给存储到code object中的consts属性了, 最后我们还需要把默认值tuple赋值到func.\_\_defaults\_\_中

**所以, 当函数有默认值的时候, 既存储在函数对象的func_defaults属性中, 也就是func.\_\_defaults\_\_, 也存储在code object(consts属性)中**

然后字节码执行的时候, 是从co_consts中获取默认值

.. code-block:: python

    In [45]: def default_func(a, b='b'):
        ...:     return a, b
        ...: 
        ...: 
    
    In [46]: dis.dis(default_func)
      2           0 LOAD_FAST                0 (a)
                  2 LOAD_FAST                1 (b)
                  4 BUILD_TUPLE              2
                  6 RETURN_VALUE
    
    In [47]: default_func.__defaults__
    Out[47]: ('b',)


在函数的\_\_defaults\_\_属性中, 存储有默认值, 但是!

看起来改变func.\_\_defaults\_\_不会影响函数的默认值, 因为执行的时候是从code_object中的consts中拿, consts在函数

定义的时候就赋值好了, 其实, 修改 **func.__defaults__** 依然会影响函数默认值

.. code-block:: python

    In [48]: default_func.__defaults__ = ('c',)
    
    In [49]: default_func('a')
    Out[49]: ('a', 'c')


我们强行改变func.\_\_defaults\_\_, 然后python也修改了函数code object中的consts, 然后影响了函数的执行


\_\_module\_\_
===============

这行变量是表示function是在哪个module中定义的, 比如

.. code-block:: python

   '''
   在test_another_func.py
   '''

   def test_another():
        print('in test_another_func module')
        return

    '''
    在test_function.py
    '''
    import test_another_func
    def test():
        print('in test_function')
        return

   print(test.__module__)
   print(test_another_func.test_another.__module__)

   '''
   然后执行python test_function.py
   然后, 输出就是
   __main__
   test_another_func
   '''




\_\_closure\_\_
=====================

func.\_\_closure\_\_, 这个变量是和内嵌函数以及作用域有关的, 简单来说, 就是

**内嵌的函数锁引用的上一层作用域的值, 存储在__closeure__中**

.. code-block:: c

    In [10]: def test():
        ...:     def nested_test():
        ...:         print('nest', spam)
        ...:         return
        ...:     spam = 'in test'
        ...:     print('in parent test')
        ...:     nested_test()
        ...:     return nested_test
        ...: 
        ...: 
        ...: 
    
    In [11]: a=test()
    in parent test
    nest in test
    
    In [12]: a
    Out[12]: <function __main__.test.<locals>.nested_test>
    
    In [13]: a.__closure__
    Out[13]: (<cell at 0x7f3bb68b2708: str object at 0x7f3bb54f5420>,)
    
    In [14]: dis.dis(a)
      3           0 LOAD_GLOBAL              0 (print)
                  2 LOAD_CONST               1 ('nest')
                  4 LOAD_DEREF               0 (spam)
                  6 CALL_FUNCTION            2
                  8 POP_TOP
    
      4          10 LOAD_CONST               0 (None)
                 12 RETURN_VALUE
    
    In [15]: a.__closure__
    Out[15]: (<cell at 0x7f3bb68b2708: str object at 0x7f3bb54f5420>,)



function对象
==============

cpython/Include/funcobject.h

.. code-block:: c

    typedef struct {
        PyObject_HEAD
        PyObject *func_code;	/* A code object, the __code__ attribute */
        PyObject *func_globals;	/* A dictionary (other mappings won't do) */
        PyObject *func_defaults;	/* NULL or a tuple */
        PyObject *func_kwdefaults;	/* NULL or a dict */
        PyObject *func_closure;	/* NULL or a tuple of cell objects */
        PyObject *func_doc;		/* The __doc__ attribute, can be anything */
        PyObject *func_name;	/* The __name__ attribute, a string object */
        PyObject *func_dict;	/* The __dict__ attribute, a dict or NULL */
        PyObject *func_weakreflist;	/* List of weak references */
        PyObject *func_module;	/* The __module__ attribute, can be anything */
        PyObject *func_annotations;	/* Annotations, a dict or NULL */
        PyObject *func_qualname;    /* The qualified name */
    
        /* Invariant:
         *     func_closure contains the bindings for func_code->co_freevars, so
         *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
         *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
         */
    } PyFunctionObject;



函数的global
================

函数的global域则是定义的时候就保存好了


执行opcode
===============

