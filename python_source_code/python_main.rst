########
main函数
########

python中main函数的流程

main函数是在cpython/Modules/main.c

.. code-block:: c

    /* Main program */
    
    int
    Py_Main(int argc, wchar_t **argv)
    {
        // 省略代码
    }

以下代码片段都是在Py_Main函数中


初始化解释器和各个内置模块
==============================

在Py_Main中, 判断完一系列的参数之后, 第一步就是调用Py_Initialize去初始化python的运行环境

而Py_Initialize会调用到_Py_InitializeEx_Private函数, 这个函数就是最主要的初始化行为

1. 包括创建主线程的tstate, 创建解释器, 加载sys, builtins等等模块

2. 初始化exception, 把exception加入到builtin字典中

3. 所有模块会赋值到解释器对象上, 比如sys模块, 就是 *interp->sysdict = PyModule_GetDict(sysmod);*

4. 并且, interp->modules被初始化为一个dict, 保存加载了哪些模块, 然后加载module前先根据module name, 判断module name是否

   存在interp->modules中, 如果存在, 则表示已经加载过了, 则直接返回, 否则新加载, 然后加入到interp->modules这个dict中

cpython/Python/pylifecircle.c

.. code-block:: c

    void
    Py_Initialize(void)
    {
        Py_InitializeEx(1);
    }

    void
    _Py_InitializeEx_Private(int install_sigs, int install_importlib)
    {
        // 解释器对象和线程状态
        PyInterpreterState *interp;
        PyThreadState *tstate;
        PyObject *bimod, *sysmod, *pstderr;
        char *p;
        extern void _Py_ReadyTypes(void);
    
        // 如果已经初始化了, 返回
        if (initialized)
            return;
        // 否则设置全局初始化变量为1
        initialized = 1;
    
        // 省略代码
    
        // 初始化随机种子
        _PyRandom_Init();
    
        // 新建一个解释器对象
        interp = PyInterpreterState_New();
        if (interp == NULL)
            Py_FatalError("Py_Initialize: can't make first interpreter");
    
        // 设置线程状态
        tstate = PyThreadState_New(interp);
        if (tstate == NULL)
            Py_FatalError("Py_Initialize: can't make first thread");
        (void) PyThreadState_Swap(tstate);
    
        // 省略代码
    
        // 下面的init函数都是各个模块的init函数
        if (!_PyFrame_Init())
            Py_FatalError("Py_Initialize: can't init frames");
    
        if (!_PyLong_Init())
            Py_FatalError("Py_Initialize: can't init longs");
    
        if (!PyByteArray_Init())
            Py_FatalError("Py_Initialize: can't init bytearray");
    
        if (!_PyFloat_Init())
            Py_FatalError("Py_Initialize: can't init float");
    
        interp->modules = PyDict_New();
        if (interp->modules == NULL)
            Py_FatalError("Py_Initialize: can't make modules dictionary");
    
        /* Init Unicode implementation; relies on the codec registry */
        // unicode模块
        if (_PyUnicode_Init() < 0)
            Py_FatalError("Py_Initialize: can't initialize unicode");
        if (_PyStructSequence_Init() < 0)
            Py_FatalError("Py_Initialize: can't initialize structseq");
    
        // 这里初始化builtin模块
        bimod = _PyBuiltin_Init();
        // 保存builtin模块到解释器
        _PyImport_FixupBuiltin(bimod, "builtins");
        interp->builtins = PyModule_GetDict(bimod);
        if (interp->builtins == NULL)
            Py_FatalError("Py_Initialize: can't initialize builtins dict");
        Py_INCREF(interp->builtins);
    
        // 这里是初始化built的异常
        /* initialize builtin exceptions */
        _PyExc_Init(bimod);
    
        // 这里就是sys模块了
        sysmod = _PySys_Init();
    
        // 下面就是保存sys模块到解释器
        if (sysmod == NULL)
            Py_FatalError("Py_Initialize: can't initialize sys");
        interp->sysdict = PyModule_GetDict(sysmod);
        if (interp->sysdict == NULL)
            Py_FatalError("Py_Initialize: can't initialize sys dict");
        Py_INCREF(interp->sysdict);
        _PyImport_FixupBuiltin(sysmod, "sys");
        // 这里是sys.path
        PySys_SetPath(Py_GetPath());
        PyDict_SetItemString(interp->sysdict, "modules",
                             interp->modules);
    
        // 导入的初始化
        _PyImport_Init();
    
        // 这里是导入hook的初始化
        _PyImportHooks_Init();
    
        // 这里是warning的初始化
        /* Initialize _warnings. */
        _PyWarnings_Init();
    
        // 下面还有time, signal等模块的初始化, 先省略吧
    
    }


run_file/PyRun_AnyFileExFlags
===================================

一般, shell模式或者直接执行py文件, 都会走到run_file这个函数, run_file直接调PyRun_AnyFileExFlags


.. code-block:: c

    int
    PyRun_AnyFileExFlags(FILE *fp, const char *filename, int closeit,
                         PyCompilerFlags *flags)
    {
        if (filename == NULL)
            filename = "???";
        if (Py_FdIsInteractive(fp, filename)) {
            int err = PyRun_InteractiveLoopFlags(fp, filename, flags);
            if (closeit)
                fclose(fp);
            return err;
        }
        else
            return PyRun_SimpleFileExFlags(fp, filename, closeit, flags);
    }

会判断是否是shell模式, 判断依据是:

.. code-block:: c

    int
    Py_FdIsInteractive(FILE *fp, const char *filename)
    {
        if (isatty((int)fileno(fp)))
            return 1;
        if (!Py_InteractiveFlag)
            return 0;
        return (filename == NULL) ||
               (strcmp(filename, "<stdin>") == 0) ||
               (strcmp(filename, "???") == 0);
    }


所以

1. 传入的filename是NULL的话, 就是shell模式

2. 传入的fp有效, 那么就是直接执行py文件


shell模式
==============

调用函数PyRun_InteractiveLoopFlags

主要是while循环, 没输入一行, 解析一行为字节码, 然后执行字节码



执行py文件模式
=================

调用函数PyRun_SimpleFileExFlags

1. 首先加载 \_\_main\_\_ 这个全局module, 因为在初始化的时候已经生成了 \_\_main\_\_模块了

   所以这里直接从interp->modules中拿

.. code-block:: c

    int
    PyRun_SimpleFileExFlags(FILE *fp, const char *filename, int closeit,
                            PyCompilerFlags *flags)
    {
        PyObject *m, *d, *v;
        const char *ext;
        int set_file_name = 0, ret = -1;
        size_t len;
    
        // 从interp->modules中拿到__main__
        m = PyImport_AddModule("__main__");
        if (m == NULL)
            return -1;
        Py_INCREF(m);
        // 拿到__main__中的可访问对象的dict
        d = PyModule_GetDict(m);
        if (PyDict_GetItemString(d, "__file__") == NULL) {
            PyObject *f;
            f = PyUnicode_DecodeFSDefault(filename);
            if (f == NULL)
                goto done;
            if (PyDict_SetItemString(d, "__file__", f) < 0) {
                Py_DECREF(f);
                goto done;
            }
            if (PyDict_SetItemString(d, "__cached__", Py_None) < 0) {
                Py_DECREF(f);
                goto done;
            }
            set_file_name = 1;
            Py_DECREF(f);
        }
        len = strlen(filename);
        ext = filename + len - (len > 4 ? 4 : 0);
        if (maybe_pyc_file(fp, filename, ext, closeit)) {
            // 运行pyc文件
            FILE *pyc_fp;
            /* Try to run a pyc file. First, re-open in binary */
            if (closeit)
                fclose(fp);
            if ((pyc_fp = _Py_fopen(filename, "rb")) == NULL) {
                fprintf(stderr, "python: Can't reopen .pyc file\n");
                goto done;
            }
    
            if (set_main_loader(d, filename, "SourcelessFileLoader") < 0) {
                fprintf(stderr, "python: failed to set __main__.__loader__\n");
                ret = -1;
                fclose(pyc_fp);
                goto done;
            }
            v = run_pyc_file(pyc_fp, filename, d, d, flags);
            fclose(pyc_fp);
        } else {
            /* When running from stdin, leave __main__.__loader__ alone */
            // 运行py文件
            if (strcmp(filename, "<stdin>") != 0 &&
                set_main_loader(d, filename, "SourceFileLoader") < 0) {
                fprintf(stderr, "python: failed to set __main__.__loader__\n");
                ret = -1;
                goto done;
            }
            // 就是这里
            v = PyRun_FileExFlags(fp, filename, Py_file_input, d, d,
                                  closeit, flags);
        }
        flush_io();
        if (v == NULL) {
            PyErr_Print();
            goto done;
        }
        Py_DECREF(v);
        ret = 0;
      done:
        if (set_file_name && PyDict_DelItemString(d, "__file__"))
            PyErr_Clear();
        Py_DECREF(m);
        return ret;
    }


所以, 运行py文件就是PyRun_FileExFlags函数



