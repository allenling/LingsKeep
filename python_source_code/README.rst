python源码实现
===============

主要是3.6版本


python接口分层
=================

简单分析python中layer的设计:


.. code-block:: python

    '''
    
        +--------------------------------------------------------+
        |                                                        |
        | 编译层                                                 |
        | 这一层主要是把pyton语法转成字节码, 然后调用对应的接口  |
        | 代码中cpython/Python/ceval.c                           |
        |                                                        |
        +--------------------------------------------------------+
        |                                                        |
        | 中间接口                                               |
        | 这一层是提供编译层和具体实现之间的一个中间层           |
        |                                                        |
        +--------------------------------------------------------+
        |                                                        |
        | 对象层, 包括对象的定义和方法的定义, 和gc, 内存管理交互 |
        |                                                        |
        +--------------------------------------------------------+
        |                            |                           |
        | gc操作                     | 内存分配和管理            |
        |                            |                           |
        +----------------------------+---------------------------+
    
    '''


启动python
==============

**更具体的流程在python_main.rst**

python的入口main函数是在cpython/Programs/main.c, 该main函数会调用Py_Main这个函数, Py_Main是在

cpython/Modules/main.c

.. code-block:: c

    int
    Py_Main(int argc, wchar_t **argv)
    {
    
        // 省略代码
    
        // 如果传入的是命令
        if (command) {
            sts = run_command(command, &cf);
            PyMem_RawFree(command);
        } else if (module) {
            // 如果是要运行module
            sts = (RunModule(module, 1) != 0);
        }else {
            // 省略代码
    
            否则运行shell模式
            if (sts == -1)
                // shell模式中, fp就是标准输入了, filename则是空
                sts = run_file(fp, filename, &cf);
        }
    
        // 省略代码
    
    }


所以, python运行是从fp读取命令然后执行的, shell模式就是标准输入

run_file
===========

.. code-block:: c

    static int
    run_file(FILE *fp, const wchar_t *filename, PyCompilerFlags *p_cf)
    {
    
        // 省略代码
    
        if (filename) {
            // 如果传入了文件名, 则解析文件名
            unicode = PyUnicode_FromWideChar(filename, wcslen(filename));
            if (unicode != NULL) {
                bytes = PyUnicode_EncodeFSDefault(unicode);
                Py_DECREF(unicode);
            }
            if (bytes != NULL)
                filename_str = PyBytes_AsString(bytes);
            else {
                PyErr_Clear();
                filename_str = "<encoding error>";
            }
        }
        else
            // 否则文件名则是<stdin>
            filename_str = "<stdin>";
        // 这里继续
        run = PyRun_AnyFileExFlags(fp, filename_str, filename != NULL, p_cf);
        Py_XDECREF(bytes);
        return run != 0;
    }

PyRun_AnyFileExFlags
=========================

.. code-block:: c

    int
    PyRun_AnyFileExFlags(FILE *fp, const char *filename, int closeit,
                         PyCompilerFlags *flags)
    {
        if (filename == NULL)
            filename = "???";
        // 下面的if是判断是否是shell模式了
        if (Py_FdIsInteractive(fp, filename)) {
            // 运行shell模式
            int err = PyRun_InteractiveLoopFlags(fp, filename, flags);
            if (closeit)
                fclose(fp);
            return err;
        }
        else
            // 执行文件
            return PyRun_SimpleFileExFlags(fp, filename, closeit, flags);
    }


PyRun_InteractiveLoopFlags
==============================

运行shell模式

.. code-block:: c

    int
    PyRun_InteractiveLoopFlags(FILE *fp, const char *filename_str, PyCompilerFlags *flags)
    {
        // 省略代码
    
        // 下面的do while循环就是一直执行代码的地方
        // while的终止条件是ret不等于E_EOF
        do {
            ret = PyRun_InteractiveOneObjectEx(fp, filename, flags);
            // ret是-1, 则退出
            if (ret == -1 && PyErr_Occurred()) {
                /* Prevent an endless loop after multiple consecutive MemoryErrors
                 * while still allowing an interactive command to fail with a
                 * MemoryError. */
                if (PyErr_ExceptionMatches(PyExc_MemoryError)) {
                    if (++nomem_count > 16) {
                        PyErr_Clear();
                        err = -1;
                        break;
                    }
                } else {
                    nomem_count = 0;
                }
                PyErr_Print();
                flush_io();
            } else {
                nomem_count = 0;
            }
            _PY_DEBUG_PRINT_TOTAL_REFS();
        } while (ret != E_EOF);
    
        // 省略代码
    
    }

其中

1. PyRun_InteractiveOneObjectEx这个函数是执行代码的过程, 然后ret是执行的结构, 所以真正解析的地方是在PyRun_InteractiveOneObjectEx里面

2. 如果在shell中输入 *exit()*, 然后ret返回的是-1, 走退出流程

PyRun_InteractiveOneObjectEx
================================

这里是获得标准输入的字符串, 解析, 然后生成codeobject, 执行codeobject

.. code-block:: c

    static int
    PyRun_InteractiveOneObjectEx(FILE *fp, PyObject *filename,
                                 PyCompilerFlags *flags)
    {
        // 省略代码

        // 拿到数据
        mod = PyParser_ASTFromFileObject(fp, filename, enc, Py_single_input, ps1, ps2, flags, &errcode, arena);
        
        // 省略代码
        
        // 执行代码
        v = run_mod(mod, filename, d, d, flags, arena);

        // 省略代码
    
    }


获取输入和语法解析调用路径:

PyRun_InteractiveOneObjectEx -> PyParser_ASTFromFileObject -> PyParser_ParseFileObject -> parsetok -> PyTokenizer_Get -> tok_get -> tok_nextc

.. code-block:: c

    static int
    tok_nextc(struct tok_state *tok)
    {
    
        // 省略代码
        
        if (tok->prompt != NULL) {
           // 调用PyOS_Readline去读取标准输入的代码
           char *newtok = PyOS_Readline(stdin, stdout, tok->prompt);
        
        // 省略代码
        // 省略的包括了解析语法
    
    }

所以, 这里有个重点: **shell模式下, 每次你输入一行, 那么python就解析一行, 而执行文件的模式也一样, 读取一行, 解析一行**

*比如shell模式下, 你要定义函数, 第一行输入 def test():, 然后回车, 那么python从PyOS_Readline中, 就得到这个语句, 然后发现是定义函数, 则
继续, 然后你输入第二行是4个空格+一个语句, 比如a = 1, 那么python再次解析这行为字节码, 然后发现当前是处于函数定义中, 则把这个a=1的字节码
加入到函数code object中的co_code中, 所以就是, 在函数定义的时候, 你每输入一行, python就把语句编译成字节码
直到你定义完函数(输入两个回车), 然后python之前生成的code object和名称给对应起来. 具体过程请参考python_function.rst*

parsetok
===============


而在parsetok中, 有:

.. code-block:: c

    static node *
    parsetok(struct tok_state *tok, grammar *g, int start, perrdetail *err_ret,
             int *flags)
    {
        for (;;) {
            // 拿到输入的内容
            type = PyTokenizer_Get(tok, &a, &b);
            if (type == ERRORTOKEN) {
                err_ret->error = tok->done;
                break;
            }
        }
    
    }

1. parse_ok, tok_get和tok_nextc主要是读取标准输入, 然后解析语法

2. 比如输入 *x=1*, 则解析之后, tok这个对象的curr属性就是: *tok->curr = "x=1\n"*


最后是run_mode

.. code-block:: c

    static PyObject *
    run_mod(mod_ty mod, PyObject *filename, PyObject *globals, PyObject *locals,
                PyCompilerFlags *flags, PyArena *arena)
    {
        PyCodeObject *co;
        PyObject *v;
        // 编译成codeobject
        co = PyAST_CompileObject(mod, filename, flags, -1, arena);
        if (co == NULL)
            return NULL;
        // 执行codeobject
        v = PyEval_EvalCode((PyObject*)co, globals, locals);
        Py_DECREF(co);
        return v;
    }



----

codeobject编译过程
=====================
            

创建codeobject
===================

接之前的函数调用路径有, run_mod -> PyAST_CompileObject -> compiler_mod -> assemble -> makecode -> PyCode_New

比如 *x[1] = 'a'* 这个代码, 执行之前, 会编译生成一个codeobject

.. code-block:: c

    // cpython/Objects/codeobject.c
    PyCodeObject *
    PyCode_New(int argcount, int kwonlyargcount,
               int nlocals, int stacksize, int flags,
               PyObject *code, PyObject *consts, PyObject *names,
               PyObject *varnames, PyObject *freevars, PyObject *cellvars,
               PyObject *filename, PyObject *name, int firstlineno,
               PyObject *lnotab)
    {
    
        // 新建的codeobject
        PyCodeObject *co;

        // 省略代码

        // 下面这些就是判断输入的consts, name等等参数了
        if (argcount < 0 || kwonlyargcount < 0 || nlocals < 0 ||
            code == NULL ||
            consts == NULL || !PyTuple_Check(consts) ||
            names == NULL || !PyTuple_Check(names) ||
            varnames == NULL || !PyTuple_Check(varnames) ||
            freevars == NULL || !PyTuple_Check(freevars) ||
            cellvars == NULL || !PyTuple_Check(cellvars) ||
            name == NULL || !PyUnicode_Check(name) ||
            filename == NULL || !PyUnicode_Check(filename) ||
            lnotab == NULL || !PyBytes_Check(lnotab) ||
            !PyObject_CheckReadBuffer(code)) {
            PyErr_BadInternalCall();
            return NULL;
        }

        /* Ensure that the filename is a ready Unicode string */
        if (PyUnicode_READY(filename) < 0)
            return NULL;

        // 下面是缓存字符串的流程, 和字符串对象的intern机制有关
        intern_strings(names);
        intern_strings(varnames);
        intern_strings(freevars);
        intern_strings(cellvars);
        intern_string_constants(consts);

        // 省略代码

        // 下面就是codeobject的创建, 赋值的过程
        co = PyObject_NEW(PyCodeObject, &PyCode_Type);
        if (co == NULL) {
            if (cell2arg)
                PyMem_FREE(cell2arg);
            return NULL;
        }
        co->co_argcount = argcount;
        co->co_kwonlyargcount = kwonlyargcount;
        co->co_nlocals = nlocals;
        co->co_stacksize = stacksize;
        co->co_flags = flags;
        Py_INCREF(code);
        // 这是是赋值字节码的地方
        co->co_code = code;
        Py_INCREF(consts);
        co->co_consts = consts;
        Py_INCREF(names);
        co->co_names = names;
        Py_INCREF(varnames);
        co->co_varnames = varnames;
        Py_INCREF(freevars);
        co->co_freevars = freevars;
        Py_INCREF(cellvars);
        co->co_cellvars = cellvars;
        co->co_cell2arg = cell2arg;
        Py_INCREF(filename);
        co->co_filename = filename;
        Py_INCREF(name);
        co->co_name = name;
        co->co_firstlineno = firstlineno;
        Py_INCREF(lnotab);
        co->co_lnotab = lnotab;
        co->co_zombieframe = NULL;
        co->co_weakreflist = NULL;
        co->co_extra = NULL;
        return co;
    
    }


运行codeobject
===================

run_mod函数运行codeobjetc中的字节码(下面代码是在shell模式下):

.. code-block:: c

    // cpython/Python/pythonrun.c
    static PyObject *
    run_mod(mod_ty mod, PyObject *filename, PyObject *globals, PyObject *locals,
                PyCompilerFlags *flags, PyArena *arena)
    {
    
        // 省略代码
        
        // 这一句就是调用上面的PyCode_New去生成返回codeobject
        co = PyAST_CompileObject(mod, filename, flags, -1, arena);
        
        // 执行codeobject
        v = PyEval_EvalCode((PyObject*)co, globals, locals);
        
        // 省略代码
    
    }

创建frame
============

执行的时候, 根据当前线程的状态和codeobject, 创建需要执行的frame, 然后执行frame

关于frame object和code object的关系嘛, 大概来说就是:

python的解释器也是一个栈执行的机器, 就是入栈, 然后出栈的过程, 入栈执行的就叫做frame, python中, 一个frame就表示了一个code object, 也就是一串字节码.

这里用frame object保存code object, 然后把frame object传给解释器

调用关系

.. code-block:: python

    '''
    
    PyEval_EvalCode -> _PyEval_EvalCodeWithName -> PyFrame_New
    
                                                -> PyEval_EvalFrameEx
    
    '''

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
    
    // 省略代码
    
        /* Create the frame */
        // 线程状态
        tstate = PyThreadState_GET();
        assert(tstate != NULL);
        // 执行的frame
        f = PyFrame_New(tstate, co, globals, locals);

        // 省略代码

        // 这里执行frame
        retval = PyEval_EvalFrameEx(f,0);
    
        // 省略代码
    
    }


执行frame
============

执行frame是使用当前解释器去执行


.. code-block:: c


    // cpython/Python/ceval.c
    PyObject *
    PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
    {
        // 当前线程状态
        PyThreadState *tstate = PyThreadState_GET();
        // 解释器对象去执行frame
        return tstate->interp->eval_frame(f, throwflag);
    }


而interp->eval_frame函数是指向(默认)_PyEval_EvalFrameDefault

.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {

        这里就是具体执行字节码的地方
        
    }

执行字节码
==============

通过dis查到 *x[1] = 'a'* 的操作码是STORE_SUBSCR:

.. code-block:: python

    In [13]: import dis
    
    In [14]: dis.dis("x[1]='a'")
      1           0 LOAD_CONST               0 ('a')
                  2 LOAD_NAME                0 (x)
                  4 LOAD_CONST               1 (1)
                  6 STORE_SUBSCR
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE

然后在_PyEval_EvalFrameDefault中:

由于之前frame object创建的时候, 把codeobject传给frame object保存起来了, 所以这里还是可以得到codeobject的

.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 当前的opcode
        int opcode;  /* Current opcode */


        // 把传入的frame object赋值到当前线程状态上保存起来
        tstate->frame = f;


        // 通过frame, 拿到codeobject和它的属性
        co = f->f_code;
        names = co->co_names;
        consts = co->co_consts;
        fastlocals = f->f_localsplus;
        freevars = f->f_localsplus + co->co_nlocals;

        // 拿到第一个字节码
        first_instr = (_Py_CODEUNIT *) PyBytes_AS_STRING(co->co_code);

        // 下一个字节码就是第一个字节码
        next_instr = first_instr;

        // 无限循环, 一段一段地去执行codeobject的字节码
        for (;;) {

            // 省略代码

            // 这一句是去拿当前的opcode的地方
            // 这个宏是通过next_instr获得opcode的
            // 并且把next_instr++
            NEXTOPARG();

            switch (opcode){

                TARGET(STORE_SUBSCR) {
                    PyObject *sub = TOP();
                    PyObject *container = SECOND();
                    PyObject *v = THIRD();
                    int err;
                    STACKADJ(-3);
                    /* container[sub] = v */
                    err = PyObject_SetItem(container, sub, v);
                    Py_DECREF(v);
                    Py_DECREF(container);
                    Py_DECREF(sub);
                    if (err != 0)
                        goto error;
                    DISPATCH();
                }
            }

            // 省略代码

        }

        // 省略代码
    }


执行中调用的接口不是具体的实现, 而是一个通用的接口, 比如PyObject_SetItem, 这个接口负责根据对象不同调用不同的实现.


中间层接口
================

中间层的接口放在cpython/Objects/abstract.c中, 比如上面的PyObject_SetItem:

.. code-block:: c


    int
    PyObject_SetItem(PyObject *o, PyObject *key, PyObject *value)
    {
        PyMappingMethods *m;
    
        if (o == NULL || key == NULL || value == NULL) {
            null_error();
            return -1;
        }
        // 先判断对象是否定义有mapping的操作
        m = o->ob_type->tp_as_mapping;
        if (m && m->mp_ass_subscript)
            return m->mp_ass_subscript(o, key, value);
    
        // 再判断对象是否定义有sequence的操作
        if (o->ob_type->tp_as_sequence) {
            if (PyIndex_Check(key)) {
                Py_ssize_t key_value;
                key_value = PyNumber_AsSsize_t(key, PyExc_IndexError);
                if (key_value == -1 && PyErr_Occurred())
                    return -1;
                return PySequence_SetItem(o, key_value, value);
            }
            else if (o->ob_type->tp_as_sequence->sq_ass_item) {
                type_error("sequence index must be "
                           "integer, not '%.200s'", key);
                return -1;
            }
        }
        
        // 没有mapping操作, 也没定义有sequence操作, 报错
        type_error("'%.200s' object does not support item assignment", o);
        return -1;
    }

所以, 这一层只是负责调用对象对应的方法而已, 具体实现交给对象本身



对象层/gc/内存管理
====================

负责实现具体的操作, 比如上面的PyObject_SetItem, 在dict对象中, 有:


.. code-block:: c

    // 这里定义了mapping操作
    PyTypeObject PyDict_Type = {
        &dict_as_mapping,                           /* tp_as_mapping */
    }
    
    // mapping的实现
    static PyMappingMethods dict_as_mapping = {
        (lenfunc)dict_length, /*mp_length*/
        (binaryfunc)dict_subscript, /*mp_subscript*/
        // 这个就是set_item的函数
        (objobjargproc)dict_ass_sub, /*mp_ass_subscript*/
    };


并且, 对象实现的时候是需要跟gc和内存管理交互的:

1. 如果对象是需要gc的对象, 那么new一个对象的时候会把新建的对象加入到gc链表中.

2. new一个对象的时候, 往往有自己的缓存, 需要自己实现, 否则直接通过内存管理接口去分配内存.

