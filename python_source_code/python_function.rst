####################
python中函数
####################

参考:

.. [1] http://www.wklken.me/posts/2015/09/04/python-source-closure.html

参考1对freevars和cellvars有比较详细的解释, 下面的freevars和cellvars是在参考1上总结的


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

在shell模式下, 你每输入一行, python就会解析该行:

比如你要定义函数, 第一行输入 def test():, 然后回车, 那么python从PyOS_Readline中就读取到这一行, 然后发现是定义函数, 则

继续, 然后你输入第二行是4个空格+一个语句, 比如a = 1, 那么python再次解析这行, 然后发现当前是函数定义, 然后继续

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


保存了很多信息, 比如qualname, defaults, code(code object)等等


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

这是因为在执行函数的时候, 调用路径是: 

call_function -> fast_function -> _PyEval_EvalCodeWithName

在fast_function中, 从function object中拿到默认参数, 也就是function.func_defaults

.. code-block:: c

    static PyObject *
    fast_function(PyObject *func, PyObject **stack,
                  Py_ssize_t nargs, PyObject *kwnames)
    {
    
        // 显然, 这个宏就是拿到function.func_defaults
        kwdefs = PyFunction_GET_KW_DEFAULTS(func);
    
    }


然后把kwdefs传入给_PyEval_EvalCodeWithName, 这样, 字节码中拿到的默认值就是修改过后的了!!!

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

函数闭包func.\_\_closure\_\_, 和内嵌函数以及作用域有关, freevars和cellvars, 参考 [1]_

  *如果发现当前函数co_cellvars非空, 即表示存在被内层函数调用的变量, 那么遍历这个co_cellvars集合, 拿到集合中每个变量名在当前名字空间中的值, 然后放到当前函数的f->f_localsplus中.*
  
  --- 参考1

所以, 主要是freevars和cellvars两个变量的处理流程, 下面的流程和参考1中的差不多, 只是流程上, 字节码变了, 比如没有了MAKE_CLOSURE

一个内联函数的例子:

.. code-block:: python

    In [2]: def test(x):
       ...:     def inner():
       ...:         print(spam)
       ...:         return
       ...:     spam = x
       ...:     return inner
       ...: 
    
    In [3]: inner_func=test('data')
    
    In [4]: inner_func()
    data

然后, 我们看看两个函数的freevars和cellvars:

.. code-block:: python

    In [8]: for i in [inner_func, test]:
       ...:     name = getattr(i, '__qualname__')
       ...:     co = getattr(i, '__code__')
       ...:     print(name, co.co_freevars, co.co_cellvars)
       ...:     
    test.<locals>.inner ('spam',) ()
    test () ('spam',)

所以, freevars表示需要从父函数中获取的变量, 而cellvars则是表示内联函数需要的变量

所以, inner_func有co_freevars, 表示其需要从父函数test中拿变量, 而父函数test有co_cellvars, 表示存在内联函数, 并且内联函数需要引用父函数的变量

接着, 看看字节码, 然后看看字节码的具体C代码


.. code-block:: python

    In [9]: dis.dis(test)
      2           0 LOAD_CLOSURE             0 (spam)
                  2 BUILD_TUPLE              1
                  4 LOAD_CONST               1 (<code object inner at 0x7f722cb69db0, file "<ipython-input-2-1cf60ee42213>", line 2>)
                  6 LOAD_CONST               2 ('test.<locals>.inner')
                  8 MAKE_FUNCTION            8
                 10 STORE_FAST               1 (inner)
    
      5          12 LOAD_FAST                0 (x)
                 14 STORE_DEREF              0 (spam)
    
      6          16 LOAD_FAST                1 (inner)
                 18 RETURN_VALUE
    
    In [10]: dis.dis(inner_func)
      3           0 LOAD_GLOBAL              0 (print)
                  2 LOAD_DEREF               0 (spam)
                  4 CALL_FUNCTION            1
                  6 POP_TOP
    
      4           8 LOAD_CONST               0 (None)
                 10 RETURN_VALUE


python经过语法解析之后, 创建code object的时候, 知道spam这个变量是需要额外处理的, 也就是调用LOAD_CLOSURE创建闭包的

当我们调用函数 *test()*, 执行的时候, 把spam的值设置到内联函数inner中的\_\_closure\_\_的, 然后在inner函数, 查找spam不是LOAD_CONST, LOAD_NAME, 而是

LOAD_DEREF, 去freevars中查找, 也就是_\_\_closure\_\_中查找, 所以找到父函数test中, spam的值了

当我们执行 *x=test('asd')* 的时候, 流程如下:

设置cellvars
----------------

调用路径是 call_function -> fast_function -> _PyEval_EvalCodeWithName

在_PyEval_EvalCodeWithName中, 会去处理cellvars, 创建一个空的cell object, 把该cell object加载到fastlocals, 也就是frame->f_localsplus中

cpython/Python/ceval.c

.. code-block:: c

    static PyObject *
    _PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
               PyObject **args, Py_ssize_t argcount,
               PyObject **kwnames, PyObject **kwargs,
               Py_ssize_t kwcount, int kwstep,
               PyObject **defs, Py_ssize_t defcount,
               PyObject *kwdefs, PyObject *closure,
               PyObject *name, PyObject *qualname)
    {
    
        // 创建frame
        f = PyFrame_New(tstate, co, globals, locals);

        // 拿到fastlocals
        fastlocals = f->f_localsplus;
        freevars = f->f_localsplus + co->co_nlocals;
    
        // 传入的co是函数test的code object
        for (i = 0; i < PyTuple_GET_SIZE(co->co_cellvars); ++i) {
            PyObject *c;
            int arg;
            /* Possibly account for the cell variable being an argument. */
            if (co->co_cell2arg != NULL &&
                (arg = co->co_cell2arg[i]) != CO_CELL_NOT_AN_ARG) {
                c = PyCell_New(GETLOCAL(arg));
                /* Clear the local copy. */
                SETLOCAL(arg, NULL);
            }
            else {
                // 会走到这里, 生成一个指向NULL的cell object
                c = PyCell_New(NULL);
            }
            if (c == NULL)
                goto fail;
            // 这个宏可以理解为, 把c加入到fastlocals这个数组中
            // 然后下标是co->nlocals + i
            SETLOCAL(co->co_nlocals + i, c);
        }


        // 最后执行frame
        retval = PyEval_EvalFrameEx(f,0);
    
    }


1. 在frame中, consts, names, freevars等等变量不是用字典, 而是用数组, 然后用偏移量去

   拿到对应子数组, 这样可以快速查找. 比如上面的SETLOCAL宏中, 
   
   co->co_nlocals是2, 然后i=0, 然后SETLOCAL就是把c设置到fastlocals[2](f->f_localsplus)中, 那么frame中,

   freevars的数组就是f->f_localsplus[2], 那么如果执行freevars[1], 就是f->f_localsplus[3]


2. 这里有一个cell object, cell object是一个数据结构, 有一个ob_ref, 表示指向的是哪个对象

   我们可以猜测, 最终cell object中的ob_ref会指向我们传入的值为asd的unicode对象


3. 然后, 这里是生成一个指向NULL的cell object, **所以cell object中的指向是在函数执行的时候赋值的!!!!!!!!!**


PyEval_EvalFrameEx则是调用_PyEval_EvalFrameDefault去执行frame中的code object, 也就是执行字节码了

首先, 我们拿到一些变量


.. code-block:: c

    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
    
    
        // 执行字节码之前
        co = f->f_code;
        names = co->co_names;
        consts = co->co_consts;
        // fastlocals赋值为f->f_localsplus
        // 而freevars则是fastlocals中, 常量之后的子数组!!!!!
        fastlocals = f->f_localsplus;
        freevars = f->f_localsplus + co->co_nlocals;
    
    }


然后执行test的字节码, 我们一个个看

LOAD_CLOSURE
----------------

.. code-block:: c

        TARGET(LOAD_CLOSURE) {
            PyObject *cell = freevars[oparg];
            Py_INCREF(cell);
            PUSH(cell);
            DISPATCH();
        }

显然, 我们从freevars, 也就是locals数组中, 常量之后的子数组, 通过传参获取需要做闭包的cell object

之前, 我们是把一个指向NULL的, 空的cell object放入到freevars[0]中, 所以, 自然, oparg=0

所以, cell就是我们之前生成的指向空的cell object, 然后PUSH入栈!!!

此时, 栈从高到低, 有:

.. code-block:: python


    '''
    
    +
    |
    |
    |
    + cell
    
    '''


接下来是BUILD_TUPLE, 其中BUILD_TUPLE则会POP一下得到cell, 然后生成一个(cell, )的元祖, 然后PUSH元祖入栈


此时, 栈从高到低, 有:

.. code-block:: python


    '''
    
    +
    |
    |
    |
    + (cell, )
    
    '''


然后, 接下来是生成内联函数的过程, 也就是LOAD_CONST, MAKE_FUNCTION


MAKE_FUNCTION
===============

MAKE_FUNCTION中, 有个关键步骤, 就是, 设置inner.\_\_closure\_\_指向我们之前LOAD_CLOSURE中的tuple的

.. code-block:: c

        TARGET(MAKE_FUNCTION) {
            // pop出函数名和code object
            PyObject *qualname = POP();
            PyObject *codeobj = POP();
            // 生成code object 
            PyFunctionObject *func = (PyFunctionObject *)
                PyFunction_NewWithQualName(codeobj, f->f_globals, qualname);

            Py_DECREF(codeobj);
            Py_DECREF(qualname);
            if (func == NULL) {
                goto error;
            }

            // 这里, 绑定inner.__closure__的地方
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

MAKE_FUNCTION中, 经过两次POP, 栈上就只有我们之前LOAD_CLOSURE中生成的, 包括cell object的tuple, (cell, )

然后, 我们把func->func_closure指向POP出来的, 也就是tuple, (cell, )

此时, 父函数的cellvars和内联函数的\_\_closure\_\_就关联上了

**但是, 此时cell是指向NULL的, 显然, 就是在test执行的过程中, 动态赋值, 也就是STORE_DEREF**

STORE_DEREF
--------------

上面, 我们把父函数和内联函数的closure关联起来了, 然后我们赋值spam的时候, 我们不是常用的STORE_FAST等字节码, 而是STORE_DEREF

STORE_DEREF是拿到cell object, 然后赋值cell object的指向(ob_ref)


先看看dis的过程:

.. code-block:: python

    5          12 LOAD_FAST                0 (x)
               14 STORE_DEREF              0 (spam)


然后是STORE_DEREF:

.. code-block:: c

        TARGET(STORE_DEREF) {
            PyObject *v = POP();
            PyObject *cell = freevars[oparg];
            PyObject *oldobj = PyCell_GET(cell);
            PyCell_SET(cell, v);
            Py_XDECREF(oldobj);
            DISPATCH();
        }


我们POP, 拿到的就是我们之前LOAD_FAST x, 也就是传入的x的值, 也就是test('asd')中, 传入的值为asd的unicode object

然, 我们设置cell->ob_ref = 'asd', 这样, 父子函数都关联上具体的值了


那么, 在内联在获取父函数的变量的时候, 是LOAD_DEREF字节码

LOAD_DEREF
---------------

然后, 我们执行内联函数, 同样, 调用路径是: 

call_function -> fast_function -> _PyEval_EvalCodeWithName

但是在_PyEval_EvalCodeWithName中, 我们这次要处理的不是cellvars, 而是freevars了

.. code-block:: c

    static PyObject *
    _PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
               PyObject **args, Py_ssize_t argcount,
               PyObject **kwnames, PyObject **kwargs,
               Py_ssize_t kwcount, int kwstep,
               PyObject **defs, Py_ssize_t defcount,
               PyObject *kwdefs, PyObject *closure,
               PyObject *name, PyObject *qualname)
    {

        // freevars是f_localsplus这个数组中
        // 除了常量之外的那部分子数组
        // 比如f_localsplus= [1, 2, 3, 4, 5, ...]
        // 如果常量个数是3, 也就是co_nlocals=3, 那么
        // freevars = [4, 5, ...]
        freevars = f->f_localsplus + co->co_nlocals
    
        for (i = 0; i < PyTuple_GET_SIZE(co->co_cellvars); ++i) {
            PyObject *c;
            int arg;
            /* Possibly account for the cell variable being an argument. */
            if (co->co_cell2arg != NULL &&
                (arg = co->co_cell2arg[i]) != CO_CELL_NOT_AN_ARG) {
                c = PyCell_New(GETLOCAL(arg));
                /* Clear the local copy. */
                SETLOCAL(arg, NULL);
            }
            else {
                c = PyCell_New(NULL);
            }
            if (c == NULL)
                goto fail;
            SETLOCAL(co->co_nlocals + i, c);
        }
    
        // 显然, 内联函数的co_freevars有值, 而co_cellvars没有值, 和父函数相反
        /* Copy closure variables to free variables */
        for (i = 0; i < PyTuple_GET_SIZE(co->co_freevars); ++i) {
            // 这里!!!拿到closure的值, 也就是我们之前生成的cell object!!!!
            PyObject *o = PyTuple_GET_ITEM(closure, i);
            Py_INCREF(o);
            freevars[PyTuple_GET_SIZE(co->co_cellvars) + i] = o;
        }
    
    
    }

1. 内联函数处理freevars, 而不是cellvars, 这个和父函数相反

2. 我们从closure中获取到之前创建的cell object, 但是这个closure怎么传进来的呢?

3. closure是在fast_function中, 调用 *closure = PyFunction_GET_CLOSURE(func);* 去获取的

   显然, PyFunction_GET_CLOSURE就是获取function object中的func_closure属性

   也就是父函数在LOAD_CLOSURE中, 创建的tuple!!!!!!!!!


4. 其中, 内联函数的co_freevars则是一个tuple, ('spam',)

   也就是, spam的值需要从freevars(f->f_localsplus中, 除了常量之外的子数组)获取, 也就是

   我们需要把cell object设置到freevars中!!!!!!!!



我们设置完变量之后, 进入到LOAD_DEREF字节码流程:

.. code-block:: c

        TARGET(LOAD_DEREF) {
            PyObject *cell = freevars[oparg];
            PyObject *value = PyCell_GET(cell);
            if (value == NULL) {
                format_exc_unbound(co, oparg);
                goto error;
            }
            Py_INCREF(value);
            PUSH(value);
            DISPATCH();
        }

我们看到, 获取父函数的变量, 就是从freevars数组中拿到的了!!!!

拿到的是我们之前生成的cell object, cell->ob_ref = unicode('asd')


