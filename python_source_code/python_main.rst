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


是否是交互模式
================


.. code-block:: c

    // 这一句判断的就是是否是开启shell模式
    stdin_is_interactive = Py_FdIsInteractive(stdin, (char *)0);
    


而判断开启shell模式的依据是

cpython/Python/pylifecircle.c


.. code-block:: c

    /*
     * The file descriptor fd is considered ``interactive'' if either
     *   a) isatty(fd) is TRUE, or
     *   b) the -i flag was given, and the filename associated with
     *      the descriptor is NULL or "<stdin>" or "???".
     */
    // 根据注释, 调用系统调用isatty判断stdin是否是关联到一个终端
    // 或者带有i标识
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


初始化解释器和各个内置模块
==============================

.. code-block:: c

   Py_Initialize();

而Py_Initialize会调用到_Py_InitializeEx_Private函数

cpython/Python/pylifecircle.c

.. code-block:: c

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


开启shell
============

.. code-block:: c

    int
    PyRun_InteractiveLoopFlags(FILE *fp, const char *filename_str, PyCompilerFlags *flags)
    {
    
    }

