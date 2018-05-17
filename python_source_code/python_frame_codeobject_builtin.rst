###########################
frame/code object/built-in
###########################

python在编译过程中的frame, code object, global, local, built-in这个几个对象的创建过程

并且涉及到python变量搜索的层级LEGB原则


built-in
============

builtin在python中表示内建的变量等等, 比如KeyError等异常, sum, dict等函数, 等等等

**然后, builtin也是一个module!!!!, 我们可以import builtins**

.. code-block:: python

    In [1]: import builtins
    
    In [2]: builtins.KeyError is KeyError
    Out[2]: True
    
    In [3]: dict is builtins.dict
    Out[3]: True


所以, 我们所说的内建的常量都是从builtins这个module中获取的!!!

而在python启动的时候, 会加载builtins, 然后生成一个builtin的字典, 里面就存储了内建的各种函数等等

初始化builtins
--------------------

在python启动的时候, 会调用到_Py_InitializeEx_Private这个函数, 去初始化各种module, 其中就是有

初始化builtins

cpython/Python/pylifecycle.c

.. code-block:: c

    void
    _Py_InitializeEx_Private(int install_sigs, int install_importlib)
    {
        // 调用builtin这个module的init函数
        bimod = _PyBuiltin_Init();

        if (bimod == NULL)
            Py_FatalError("Py_Initialize: can't initialize builtins modules");
        _PyImport_FixupBuiltin(bimod, "builtins");

        // 把内建module设置到解释器上
        // 注意, 这是一个dict
        interp->builtins = PyModule_GetDict(bimod);
        if (interp->builtins == NULL)
            Py_FatalError("Py_Initialize: can't initialize builtins dict");
        Py_INCREF(interp->builtins); 
    }


globals
===========

globals是一个函数, 自然, 也是builtins中的一员

.. code-block:: python

    In [30]: builtins.globals is globals
    Out[30]: True


比如, 当我们在shell模式下, python启动的时候, globals则是__main__这个module的可访问对象的字典了, 调用路径是

.. code-block:: python

    '''
    
    main -> Py_Main -> run_file -> PyRun_AnyFileExFlags -> PyRun_InteractiveLoopFlags -> PyRun_InteractiveOneObjectEx
    
    '''


cpython/Python/pythonrun.c

.. code-block:: c

    static int
    PyRun_InteractiveOneObjectEx(FILE *fp, PyObject *filename,
                                 PyCompilerFlags *flags)
    {
    
    
    // 拿到__main__这个模块
    // module_name是值为__main__的unicode object
    mod_name = _PyUnicode_FromId(&PyId___main__);
    
    // 拿到__main__模块的PyModuleObject
    // m就是代表__main__的PyModuleObject
    m = PyImport_AddModuleObject(mod_name);
    
    // 拿到__main__中的md_dict, 可访问对象的字典
    d = PyModule_GetDict(m);
    
    // 调用run_mod去循环执行输入的语句
    // 其中第三第四个参数就是globals和locals
    v = run_mod(mod, filename, d, d, flags, arena);
    
    }


run_mod的第三第四个参数是globals和locals, 所以, shell模式下, 每次执行字节码的时候, globals就是\_\_main\_\_模块的md_dict

那么, __main__是在哪里初始化的呢?
-------------------------------------

是在python启动的时候, 调用_Py_InitializeEx_Private函数去初始化python运行参数, 比如解释器, tstate等等的时候

会在_Py_InitializeEx_Private中, 调用initmain(interp), 去初始化\_\_main\_\_这个module

cpython/Python/pylifecycle.c

.. code-block:: c

    void
    _Py_InitializeEx_Private(int install_sigs, int install_importlib)
    {
    
        initmain(interp); /* Module __main__ */
    
    }

initmain函数就是新建一个\_\_main\_\_ module

.. code-block:: c

    static void
    initmain(PyInterpreterState *interp)
    {
        PyObject *m, *d, *loader, *ann_dict;
        // 你看!!!
        // 这里就是加载__main__这个module
        m = PyImport_AddModule("__main__");
        if (m == NULL)
            Py_FatalError("can't create __main__ module");

        // 你看!!!!
        // 这里d就是__main__中的可访问对象的dict了
        // 当然, d现在是只有一些内置的元素
        // 比如__name__='__main__', ___doc__=None等等
        d = PyModule_GetDict(m);

        // 下面是初始化其他变量吧
        ann_dict = PyDict_New();
        if ((ann_dict == NULL) ||
            (PyDict_SetItemString(d, "__annotations__", ann_dict) < 0)) {
            Py_FatalError("Failed to initialize __main__.__annotations__");
        }
        Py_DECREF(ann_dict);
        if (PyDict_GetItemString(d, "__builtins__") == NULL) {
            PyObject *bimod = PyImport_ImportModule("builtins");
            if (bimod == NULL) {
                Py_FatalError("Failed to retrieve builtins module");
            }
            if (PyDict_SetItemString(d, "__builtins__", bimod) < 0) {
                Py_FatalError("Failed to initialize __main__.__builtins__");
            }
            Py_DECREF(bimod);
        }
        /* Main is a little special - imp.is_builtin("__main__") will return
         * False, but BuiltinImporter is still the most appropriate initial
         * setting for its __loader__ attribute. A more suitable value will
         * be set if __main__ gets further initialized later in the startup
         * process.
         */
        loader = PyDict_GetItemString(d, "__loader__");
        if (loader == NULL || loader == Py_None) {
            PyObject *loader = PyObject_GetAttrString(interp->importlib,
                                                      "BuiltinImporter");
            if (loader == NULL) {
                Py_FatalError("Failed to retrieve BuiltinImporter");
            }
            if (PyDict_SetItemString(d, "__loader__", loader) < 0) {
                Py_FatalError("Failed to initialize __main__.__loader__");
            }
            Py_DECREF(loader);
        }
    }

\_\_main\_\_->md_dict['\_\_name\_\_']就是'\_\_main\_\_'了

我们来看看shell模式下加的\_\_name\_\_:

.. code-block:: python

    In [36]: __name__
    Out[36]: '__main__'


因为代码上, 我们看到把\_\_main\_\_加入到了interp->modules中, 所以, 我们可以导入\_\_main\_\_

.. code-block:: c

    In [42]: import __main__
    
    In [43]: __main__
    Out[43]: <module '__main__'>
    
    In [44]: __main__?
    Type:        module
    String form: <module '__main__'>
    Docstring:   Automatically created module for IPython interactive environment

    In [45]: __main__.__name__
    Out[45]: '__main__'


而如果我们直接运行py文件的话, 传入的globals和locals也是\_\_main\_\_->md_dict

直接运行py文件的时候, 调用路径是

.. code-block:: python

    '''
    
    run_file -> PyRun_AnyFileExFlags -> PyRun_FileExFlags
    
    '''

在PyRun_FileExFlags中, 也是把\_\_main\_\_->md_dict作为globals, locals

.. code-block:: c

    int
    PyRun_SimpleFileExFlags(FILE *fp, const char *filename, int closeit,
                            PyCompilerFlags *flags)
    {
    
        m = PyImport_AddModule("__main__");
        
        d = PyModule_GetDict(m);
        
        v = PyRun_FileExFlags(fp, filename, Py_file_input, d, d, closeit, flags);
    
    
    }

code object
=================






frame
==========


在经过上面的流程, 创建了code object之后, 我们需要创建frame object

frame是python中执行的单位, 每一个frame object对应一个code object

假设在执行py文件下, 则调用路径就是

.. code-block:: python

    '''
    
    run_file -> PyRun_SimpleFileExFlags -> PyRun_AnyFileExFlags -> PyRun_FileExFlags -> run_mod
    
    -> PyEval_EvalCode -> PyEval_EvalCodeEx -> _PyEval_EvalCodeWithName
    
    '''

其中, globals和locals则会从PyRun_SimpleFileExFlags中一直传递, 在_PyEval_EvalCodeWithName中, 回去新建frame, 然后直接frame

cpython/Python/pythonrun.c

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
    
    
        // 拿到tstate, 然后创建frame
        tstate = PyThreadState_GET();
        assert(tstate != NULL);
        f = PyFrame_New(tstate, co, globals, locals);
        if (f == NULL) {
            return NULL;
        }
    
        // 还有一堆流程
    
        // 执行frame
        retval = PyEval_EvalFrameEx(f,0);
    
    }


而在PyFrame_New中, 简单点就是:

1. 赋值f->globals, f->f_locals = globals, locals, 然后f->f_builtins = builtins;

   当然, builtins是从globals中拿到的 *builtins = _PyDict_GetItemId(globals, &PyId___builtins__);*


2. 赋值code object的属性到frame, 比如 *f->f_code = code;* 等等



LEGB
=======

LEGB的原则其实是在特定条件下的, 一般都是在module中如果有执行语句, 那么才会有LEGB

如果是函数中的话, python中的编译会聪明地挑选合适的字节码, 一般很少有会直接LEGB的


比如, 我们在shell模式下, 输入一行命令, 然后python执行, 其实和我们python xxx.py一样, 只不过我们是动态添加内容到xxx.py的

比如xxx.py执行import yyy.py, 如果yyy.py中有直接执行的语句, 比如 *if __name__ == __main__* , 的话, \_\_name\_\_也是LEGB原则查找

然后在shell模式和module执行语句下, 比如 *print(__name__)*, 那么字节码就是LOAD_NAME:

.. code-block:: python

    In [6]: dis.dis("print(__name__)")
      1           0 LOAD_NAME                0 (print)
                  2 LOAD_NAME                1 (__name__)
                  4 CALL_FUNCTION            1
                  6 RETURN_VALUE

来看看LOAD_NAME的执行过程:

.. code-block:: c

        TARGET(LOAD_NAME) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *locals = f->f_locals;
            PyObject *v;
            // 先找f->f_locals
            // 不如f->f_locals为NULL, 直接报错
            if (locals == NULL) {
                PyErr_Format(PyExc_SystemError,
                             "no locals when loading %R", name);
                goto error;
            }
            // 如果f->f_locals是字典
            // 拿值
            if (PyDict_CheckExact(locals)) {
                v = PyDict_GetItem(locals, name);
                Py_XINCREF(v);
            }
            // 否则, 调用一般性的getitem函数去获取值
            else {
                v = PyObject_GetItem(locals, name);
                if (v == NULL) {
                    if (!PyErr_ExceptionMatches(PyExc_KeyError))
                        goto error;
                    PyErr_Clear();
                }
            }

            // 经过f->f_locals查找, 没找到
            if (v == NULL) {
                // 那么先找f->f_globals
                v = PyDict_GetItem(f->f_globals, name);
                Py_XINCREF(v);
                // 如果在f->f_globals中也没找到
                if (v == NULL) {
                    // 查找f->f_builtins
                    if (PyDict_CheckExact(f->f_builtins)) {
                        // 如果f->f_builtins也没找到
                        // 说明变量未定义, 报错!!!!!!!!!
                        v = PyDict_GetItem(f->f_builtins, name);
                        if (v == NULL) {
                            format_exc_check_arg(
                                        PyExc_NameError,
                                        NAME_ERROR_MSG, name);
                            goto error;
                        }
                        Py_INCREF(v);
                    }
                    else {
                        // 这里调动通用的getitem函数, 流程和上面一样
                        v = PyObject_GetItem(f->f_builtins, name);
                        if (v == NULL) {
                            if (PyErr_ExceptionMatches(PyExc_KeyError))
                                format_exc_check_arg(
                                            PyExc_NameError,
                                            NAME_ERROR_MSG, name);
                            goto error;
                        }
                    }
                }
            }
            PUSH(v);
            DISPATCH();
        }


所以, LOAD_NAME就是LEGB原则的查找


那如果我们是定义函数的呢:

.. code-block:: python

    In [6]: dis.dis("print(__name__)")
      1           0 LOAD_NAME                0 (print)
                  2 LOAD_NAME                1 (__name__)
                  4 CALL_FUNCTION            1
                  6 RETURN_VALUE
    
    In [7]: def test():
       ...:     print(__name__)
       ...:     return
       ...: 
    
    In [8]: dis.dis(test)
      2           0 LOAD_GLOBAL              0 (print)
                  2 LOAD_GLOBAL              1 (__name__)
                  4 CALL_FUNCTION            1
                  6 POP_TOP
    
      3           8 LOAD_CONST               0 (None)
                 10 RETURN_VALUE
    


我们看到, 字节码是LOAD_GLOBAL, 从字面意思上就知道, 直接从globals中拿, 而不走所谓的LEGB

如果是内联函数呢? 内联函数的字节码则是LOAD_DEREF, 这个和函数的cell object有关, 但是并不是走LEGB!!!


.. code-block:: python

    In [11]: def test_outer():
        ...:     def test_inner():
        ...:         print('inner', spam)
        ...:         return
        ...:     spam = 'spam'
        ...:     print('outer', spam)
        ...:     test_inner()
        ...:     return test_inner
        ...: 
        ...: 
    
    In [12]: inner_func = test_outer()
    outer spam
    inner spam
    
    In [13]: dis.dis(inner_func)
      3           0 LOAD_GLOBAL              0 (print)
                  2 LOAD_CONST               1 ('inner')
                  4 LOAD_DEREF               0 (spam)
                  6 CALL_FUNCTION            2
                  8 POP_TOP
    
      4          10 LOAD_CONST               0 (None)
                 12 RETURN_VALUE


**所以, 并不是所有的变量查找都是LEGB, 因为python的编译器能聪明地识别去应该去哪里找变量**

LEGB过程一般是字节码LOAD_NAME, 而LOAD_NAME则是在module中(包括shell模式, 也是执行module)

执行查找变量语句(注意, 定义语句不算)的时候, 会调用到LOAD_NAME

比如在shell模式下, *print(__name__)*, 则就需要查找 *__name__*, 那么就走LOAD_NAME

