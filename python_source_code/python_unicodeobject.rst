unicode
==========

1. python3中, str和unicode合并了, 只存在unicode

2. unicode的缓存(intern), is操作, intern手动缓存, 编译缓存和优化

3. 缓存长度为20其实是, 使用\*重复20次以上的, python不会编译时求值, 而是运行时求值

3. find算法(BM和Horspool的综合)


缓存参考: http://guilload.com/python-string-interning/


----

字符串编译流程
==================

当输入 *x='12你们'* 的时候, 经过一系列语法解析之后判断到这是一个赋值语句, 赋值的是字符串, 调用到:

// cpython/Python/ast.c

.. code-block:: c

    static int
    parsestr(struct compiling *c, const node *n, int *bytesmode, int *rawmode,
             PyObject **result, const char **fstr, Py_ssize_t *fstrlen)
    {
    
        // 代码省略

        // 拿到s, 也就是我们传入的字符串的长度
        // 调用strlen, 比如'12你们'的长度就是8
        len = strlen(s);
    
        /* Not an f-string. */
        /* Avoid invoking escape decoding routines if possible. */
        // 不是f-string模式, f-string是3.7的特性
        *rawmode = *rawmode || strchr(s, '\\') == NULL;
        // bytesmode判断是否是byte, 比如b'123'
        if (*bytesmode) {
            /* Disallow non-ASCII characters. */
            const char *ch;
            for (ch = s; *ch; ch++) {
                if (Py_CHARMASK(*ch) >= 0x80) {
                    ast_error(c, n, "bytes can only contain ASCII "
                              "literal characters.");
                    return -1;
                }
            }
            if (*rawmode)
                *result = PyBytes_FromStringAndSize(s, len);
            else
                *result = decode_bytes_with_escapes(c, n, s, len);
        } else {
            // 不是byte, 是原生字符串
            if (*rawmode)
                // 调用到这里, 返回PyUnicodeObject
                *result = PyUnicode_DecodeUTF8Stateful(s, len, NULL, NULL);
            else
                *result = decode_unicode_with_escapes(c, n, s, len);
        }
    
    
    }

该函数中, s就是我们的字符串, len则是调用strlen得到的字符串的长度, 所以s='12你们', len=8

cpython/Objects/unicodeobject.c

.. code-block:: c

    PyObject *
    PyUnicode_DecodeUTF8Stateful(const char *s,
                                 Py_ssize_t size,
                                 const char *errors,
                                 Py_ssize_t *consumed)
    {
    
        // 这里!!!!!把writer->buffer赋值为一个新的PyUnicodeObject
        if (_PyUnicodeWriter_Prepare(&writer, writer.min_length, 127) == -1)
            goto onError;
        
        // 初始化writer
        _PyUnicodeWriter_Init(&writer);
        writer.min_length = size;
        // 这里
        if (_PyUnicodeWriter_Prepare(&writer, writer.min_length, 127) == -1)
            goto onError;

        // 这里拿到s中第一个非ascii字符的位置, 并且把ascii复制到writer.data中
        // 比如'12你们', 那么pos=2
        // 所以下面的while就是从第一个非ascii字符开始去遍历
        writer.pos = ascii_decode(s, end, writer.data);
        // 所以, pos=2, s移动两个位置, 比如'12你们', 也就是s现在指向'你'
        s += writer.pos;
        
        // 一个字符一个字符去编码和存储unicode
        while (s < end) {
        
                Py_UCS4 ch;
                int kind = writer.kind;
        
                // 判断每一个字符, 注意的是每一个字符!!!!!!!
                // 下面的asciilib_函数则负责存储
                if (kind == PyUnicode_1BYTE_KIND) {
                    if (PyUnicode_IS_ASCII(writer.buffer))
                        ch = asciilib_utf8_decode(&s, end, writer.data, &writer.pos);
                    else
                        ch = ucs1lib_utf8_decode(&s, end, writer.data, &writer.pos);
                } else if (kind == PyUnicode_2BYTE_KIND) {
                    ch = ucs2lib_utf8_decode(&s, end, writer.data, &writer.pos);
                } else {
                    assert(kind == PyUnicode_4BYTE_KIND);
                    ch = ucs4lib_utf8_decode(&s, end, writer.data, &writer.pos);
                }
        
        }
    
    }


_PyUnicodeWriter_Prepare
====================================

这个函数调用的是_PyUnicodeWriter_PrepareInternal

cpython/Objects/unicodeobject.c

.. code-block:: c

    int
    _PyUnicodeWriter_PrepareInternal(_PyUnicodeWriter *writer,
                                     Py_ssize_t length, Py_UCS4 maxchar)
    {
        // 传入的length是字符串的长度, maxchar传入的是默认的ascii码最大127
        Py_ssize_t newlen;
        PyObject *newbuffer;
    
        assert(maxchar <= MAX_UNICODE);
    
        /* ensure that the _PyUnicodeWriter_Prepare macro was used */
        assert((maxchar > writer->maxchar && length >= 0)
               || length > 0);
    
        if (length > PY_SSIZE_T_MAX - writer->pos) {
            PyErr_NoMemory();
            return -1;
        }
        // writer->pos被初始化为0
        newlen = writer->pos + length;
    
        // 这里判断一下, 不过基本没什么用, 除非第一个字符串就是unicode
        maxchar = Py_MAX(maxchar, writer->min_char);
    
        // 初始化的writer->buffer是NULL
        if (writer->buffer == NULL) {
            assert(!writer->readonly);
            if (writer->overallocate
                && newlen <= (PY_SSIZE_T_MAX - newlen / OVERALLOCATE_FACTOR)) {
                /* overallocate to limit the number of realloc() */
                newlen += newlen / OVERALLOCATE_FACTOR;
            }
            if (newlen < writer->min_length)
                newlen = writer->min_length;

            // !!!!!!!所以, 我们这里调用PyUnicode_New生成一个PyUnicodeObject
            writer->buffer = PyUnicode_New(newlen, maxchar);
            if (writer->buffer == NULL)
                return -1;
        }else if () {
            // 代码省略
        }else if () {
            // 代码省略
        }
        _PyUnicodeWriter_Update(writer);
    
    }

关于PyUnicode_New, 这里是生成一个平台相关

所以, 该函数就是把writer->buffer初始化一个ascii类型的PyUnicodeObject

ascii_decode
==============

这个函数是PyUnicode_DecodeUTF8Stateful中, 调用_PyUnicodeWriter_Prepare去初始化writer之后

计算第一个非ascii字符位置, 并且把第一个非ascii字符之前的字符赋值到writer->data中

cpython/Objects/unicodeobject.c

.. code-block:: c

    static Py_ssize_t
    ascii_decode(const char *start, const char *end, Py_UCS1 *dest)
    {
        // 其中, 传入的dest是writer->data
        // start就是我们的字符串, '12你们'
        // end就是结束符
        // p指向start
        const char *p = start;
    
        // 代码省略
    
        while (p < end) {
            // 这里一个字符一个字符串去判断
            /* Fast path, see in STRINGLIB(utf8_decode) in stringlib/codecs.h
               for an explanation. */
            if (_Py_IS_ALIGNED(p, SIZEOF_LONG)) {
                /* Help allocation */
                const char *_p = p;
                while (_p < aligned_end) {
                    unsigned long value = *(unsigned long *) _p;
                    if (value & ASCII_CHAR_MASK)
                        break;
                    _p += SIZEOF_LONG;
                }
                p = _p;
                if (_p == end)
                    break;
            }
            // 这里0x80就是128, 也就是是否是小于等于127的字符, 也就是是否是ascii字符
            if ((unsigned char)*p & 0x80)
                // 如果不是ascii字符, 退出
                break;
            ++p;
        }
        // 复制第一个非ascii字符之前的内容到dest, 也就是writer->data
        memcpy(dest, start, p - start);
        // 返回位置
        return p - start;
    
    }

所以这函数是, 找到第一个非ascii字符, 复制该字符之前的ascii字符到writer->data, 返回第一个非ascii字符的位置


继续PyUnicode_DecodeUTF8Stateful
==================================

接着继续看PyUnicode_DecodeUTF8Stateful函数


.. code-block:: c

    PyObject *
    PyUnicode_DecodeUTF8Stateful(const char *s,
                                 Py_ssize_t size,
                                 const char *errors,
                                 Py_ssize_t *consumed)
    {
    
        writer.pos = ascii_decode(s, end, writer.data);
        s += writer.pos;
        while (s < end) {
    
            Py_UCS4 ch;
            int kind = writer.kind;
    
            // 由于writer被初始化为一个ascii对象, 所以对第一个
            // 非ascii字符处理的时候, 走第一个if分支
            // 第一个unicode字符之后的字符, 走其他分支
            // 比如'们'这个字符走PyUnicode_2BYTE_KIND这个分支
            if (kind == PyUnicode_1BYTE_KIND) {
                if (PyUnicode_IS_ASCII(writer.buffer))

                    // 去decode(编码), 赋值字符到writer.data
                    ch = asciilib_utf8_decode(&s, end, writer.data, &writer.pos);
                else
                    ch = ucs1lib_utf8_decode(&s, end, writer.data, &writer.pos);
            } else if (kind == PyUnicode_2BYTE_KIND) {
                ch = ucs2lib_utf8_decode(&s, end, writer.data, &writer.pos);
            } else {
                assert(kind == PyUnicode_4BYTE_KIND);
                ch = ucs4lib_utf8_decode(&s, end, writer.data, &writer.pos);
            }

            switch (ch) {
                case 0:
                    // case=0是表示已经是最后一个字符了
                    if (s == end || consumed)
                        goto End;
                    errmsg = "unexpected end of data";
                    startinpos = s - starts;
                    endinpos = end - starts;
                    break;
                case 1:
                    errmsg = "invalid start byte";
                    startinpos = s - starts;
                    endinpos = startinpos + 1;
                    break;
                case 2:
                case 3:
                case 4:
                    errmsg = "invalid continuation byte";
                    startinpos = s - starts;
                    endinpos = startinpos + ch - 1;
                    break;
                // unicode走这里!!!!!!!!!
                default:
                    if (_PyUnicodeWriter_WriteCharInline(&writer, ch) < 0)
                        goto onError;
                    continue;
            }
    
        }
        End:
        if (consumed)
            *consumed = s - starts;

        Py_XDECREF(error_handler_obj);
        Py_XDECREF(exc);
        return _PyUnicodeWriter_Finish(&writer);
    
    }

所以接下来的流程的关键是几个字符串decode的库, asciilib_utf8_decode等等, 这几个函数大同小异, 都是对unicode进行编码, 然后返回字符的unicode值

比如'你'这个字符, 经过asciilib_uf8_decode编码之后, 使用三个字节去存储该字符, 返回的ch则是'你'这个字符的unicode值, 也就是20320, 当然也是复制到writer->data

decode的流程涉及到unicode, utf8的编码和转码, 比如 *你* 这个字, 值是20320, 但是存储的时候, 是用三个字节存储[x, y, z], 但是python内部对于'你们’是使用两个字节存储, 这部分先略过

在接下来的switch语句, 走default分支, 调用_PyUnicodeWriter_WriteCharInline函数


cpython/Objects/unicodeobject.c

.. code-block:: c

    static inline int
    _PyUnicodeWriter_WriteCharInline(_PyUnicodeWriter *writer, Py_UCS4 ch)
    {
        assert(ch <= MAX_UNICODE);

        // 这里再次调用Prepare函数, 注意的是, 传入的ch是unicode字符
        // 比如'你'这个字符, 所以writer的buffer就变为compact类型的PyUnicodeObject
        if (_PyUnicodeWriter_Prepare(writer, 1, ch) < 0)
            return -1;
        PyUnicode_WRITE(writer->kind, writer->data, writer->pos, ch);
        // 然后指向下一个字符
        writer->pos++;
        return 0;
    }

这个函数再次调用_PyUnicodeWriter_Prepare去重新设置writer->buffer

.. code-block:: c

    int
    _PyUnicodeWriter_PrepareInternal(_PyUnicodeWriter *writer,
                                     Py_ssize_t length, Py_UCS4 maxchar)
    {
    
        // 代码省略
        if () {
        }else if () {
        }
        else if (maxchar > writer->maxchar) {
            // 走这个分支
            assert(!writer->readonly);
            // 新建一个buffer
            newbuffer = PyUnicode_New(writer->size, maxchar);
            if (newbuffer == NULL)
                return -1;
            // 把writer->buffer复制到newbuffer
            _PyUnicode_FastCopyCharacters(newbuffer, 0,
                                          writer->buffer, 0, writer->pos);
            // 设置writer->buffer指向newbuffer
            Py_SETREF(writer->buffer, newbuffer);
        }
    }

因为之前writer->max_char是127, 也就是writer->buffer默认是ascii类型的PyUnicodeObject, 这次要变成compact类型的PyUnicodeObject

所以调用PyUnicode_New去新建一个compact类型的PyUnicodeObject

然后继续处理'们‘这个字符, 编码, 把该字符赋值到writer->data


最后调用_PyUnicodeWriter_Finish
==================================

cpython/Objects/unicodeobject.c

这个函数基本上是说, 处理writer->buffer, 返回一个正确的unicode对象.

比如之前writer->buffer指向的PyUnicodeObject中, length=8, 这个8是strlen返回的, 但是我们调用len的时候, 应该是4, 也就是writer->pos的值

也就是length != pos, 表示有unicode, 所以finish函数就是把length = pos, 当然还是其他操作


.. code-block:: c

    int
    _PyUnicodeWriter_WriteLatin1String(_PyUnicodeWriter *writer,
                                       const char *str, Py_ssize_t len)
    {
        PyObject *str;
    
        if (writer->pos == 0) {
            Py_CLEAR(writer->buffer);
            _Py_RETURN_UNICODE_EMPTY();
        }
    
        // 拿到writer->buffer
        str = writer->buffer;
        // 然后writer->buffer被清空
        writer->buffer = NULL;
    
        if (writer->readonly) {
            assert(PyUnicode_GET_LENGTH(str) == writer->pos);
            return str;
        }
    
        // 然后这里, length不等于pos
        if (PyUnicode_GET_LENGTH(str) != writer->pos) {
            PyObject *str2;
            // 重新创建一个compact的unicodeobject
            str2 = resize_compact(str, writer->pos);
            if (str2 == NULL) {
                Py_DECREF(str);
                return NULL;
            }
            // 指向新的对象
            str = str2;
        }
    
    }


resize_compact则是重新分配大小, 先略过

小结
======

1. 初始化writer, writer->buffer先默认是一个ascii字符串, 创建为一个ascii类型的PyUnicodeObject

2. 找到第一个非ascii字符, 把该字符之前的字符都复制到writer->data, 比如例子的中'12'

3. 然后一个接一个字符去decode(编码), 编码的同时把unicode复制到writer->data中, 同时把writer->buffer变为一个compact类型的PyUnicodeObject

4. 最后finish的时候, 设置正确的length

5. 存储的时候, 一旦有unicode, 就会变成n个字节存储一个字符, 比如'12你们', '你们'都是用两个字节, 那么本来'12'都是用一个字节, 最后也会改为两个字节,

   也就是说存储会变为: ['1', '', '2', '', 'x1', 'x2', 'y1', 'y2]


获取unicode的字符串
=======================

事实上unicode的的data并没有存储在PyUnicodeObject中, 首先在debug的过程中, 我们看到都是把字符串复制到writer->data中

就算最后的finish过程, 也只是把str的大小设置成合理的大小, 但是设置大小的这一步比较关键!!!!

1. writer->data的地址是: 0x7ffff6bcee18

2. unicode的地址是: 0x7ffff6bcedc0

来看看打印字符串的时候, 去哪里拿到数据数组, 下面是unicode的repr的函数

.. code-block:: c

    static PyObject *
    unicode_repr(PyObject *unicode)
    {
    
        idata = PyUnicode_DATA(unicode);
        
        for (i = 0; i < isize; i++) {
            Py_UCS4 ch = PyUnicode_READ(ikind, idata, i);
        }
    
    }

当debug的时候, 传入的unicode的地址是0x7ffff6bcedc0, 正好是创建时候的字符串'12你们'的地址, 然后PyUnicode_DATA是一个宏


.. code-block:: c

    #define PyUnicode_DATA(op) \
        (assert(PyUnicode_Check(op)), \
         PyUnicode_IS_COMPACT(op) ? _PyUnicode_COMPACT_DATA(op) :   \
         _PyUnicode_NONCOMPACT_DATA(op))

如果是compact类型的unicode, 那么去调用_PyUnicode_COMPACT_DATA

.. code-block:: c

    /* Return a void pointer to the raw unicode buffer. */
    #define _PyUnicode_COMPACT_DATA(op)                     \
        (PyUnicode_IS_ASCII(op) ?                   \
         ((void*)((PyASCIIObject*)(op) + 1)) :              \
         ((void*)((PyCompactUnicodeObject*)(op) + 1)))

注释上说, 返回一个元素的unicode的buffer, 然后调用的是 *((void\*)((PyCompactUnicodeObject*)(op) + 1)))*, 也就是说是unicode的下一个地址

然后在debug中看到, idata的地址正好是创建的时候, writer-data的地址: 0x7ffff6bcee18, idata的接下去的地址是:

.. code-block:: python

    '''
    (char *)idata   + 0   0x7ffff6bcee18 "1"
    ((char *)idata) + 1   0x7ffff6bcee19 ""
    ((char *)idata) + 2   0x7ffff6bcee1a "2"
    ((char *)idata) + 3   0x7ffff6bcee1b ""
    ((char *)idata) + 4   0x7ffff6bcee1c "`OìN"
    ((char *)idata) + 5   0x7ffff6bcee1d "OìN"
    ((char *)idata) + 6   0x7ffff6bcee1e "ìN""
    ((char *)idata) + 7   0x7ffff6bcee1f "N"
    ((char *)idata) + 8   0x7ffff6bcee20 ""
    '''

而PyUnicode_READ读取的结果则是:

.. code-block:: python

    '''

    ((Py_UCS2 *)idata)[0] = 49(1)
    ((Py_UCS2 *)idata)[1] = 50(2)
    ((Py_UCS2 *)idata)[2] = 20320(你)
    ((Py_UCS2 *)idata)[3] = 20204(们)

    '''


PyUnicodeObject
===================

python中, 如果全都是ascii字符, 那么会返回PyASCIIObject, 如果有unicode, 那么返回compact类型的unicode对象PyCompactUnicodeObject

具体的数据结构先省略


缓存机制
===================

is操作的区别

.. code-block:: python

    In [124]: x='foo!'
    
    In [125]: y='foo!'
    
    In [126]: x is y
    Out[126]: False
    
    In [127]: x='awd'
    
    In [128]: y='awd'
    
    In [129]: x is y
    Out[129]: True


在编译中看看foo!和awd的区别, **每一个语句都会编译成一个codeobject, 每一个codeobject都有自己的consts常量**, 然后其中常量会保存在codeobject.consts中

同时, 常量(满足条件的)会被加入到全局intern这个dict里面


.. code-block:: c

    PyCodeObject *
    PyCode_New(int argcount, int kwonlyargcount,
               int nlocals, int stacksize, int flags,
               PyObject *code, PyObject *consts, PyObject *names,
               PyObject *varnames, PyObject *freevars, PyObject *cellvars,
               PyObject *filename, PyObject *name, int firstlineno,
               PyObject *lnotab)
    {
    
        // 省略代码
        
        // 这里回去操作consts
        intern_string_constants(consts);
        
        // 省略代码
    
    }

而intern_string_constants会去根据一定的规则去缓存unicode


cpython/Objects/codeobject.c

.. code-block:: c

    static int
    intern_string_constants(PyObject *tuple)
    {
        // 省略代码

        // all_name_chars则是缓存的判断条件
        if (all_name_chars(v)) {
          // 省略代码
          PyUnicode_InternInPlace(&v);
        }
        // 省略代码
    }

all_name_chars
=================

all_name_chars是判断一个字符是否需要缓存的地方

.. code-block:: c

    #define NAME_CHARS \
        "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"

    /* all_name_chars(s): true iff all chars in s are valid NAME_CHARS */
    static int
    all_name_chars(PyObject *o)
    {
        static char ok_name_char[256];
        static const unsigned char *name_chars = (unsigned char *)NAME_CHARS;
        const unsigned char *s, *e;
    
        // 非ascii码字符串不缓存
        if (!PyUnicode_IS_ASCII(o))
            return 0;
    
        // 这里是初始化过程, 所有的NAME_CHARS的字符, 在ok_name_char中都需要置1
        if (ok_name_char[*name_chars] == 0) {
            const unsigned char *p;
            for (p = name_chars; *p; p++)
                ok_name_char[*p] = 1;
        }
        s = PyUnicode_1BYTE_DATA(o);
        e = s + PyUnicode_GET_LENGTH(o);
        // 下面的循环会一个字符一个字符去判断是否
        // 是常规字符
        // e是最后一个字符, s是从第一个字符开始
        while (s != e) {
            // *s就是当前位置的字符
            if (ok_name_char[*s++] == 0)
                return 0;
        }
        return 1;
    }

其中判断流程是:

1. 非ascii字符不缓存, 比如'abc我'带有中文就不会缓存了

2. 创建长度为256的数组ok_name_char

3. 拿到常规字符串NAME_CHARS, 也就是0-9, 26个字母的大小写,也就是所有的长度为0和1的字符都默认被缓存掉了

4. 然后把NAME_CHARS在ok_name_char的位置设置为1

5. 逐个字符去判断是否是常规字符, 也就是其ok_name_char中缓存位是否是１, 如果是0, 退出

所以:

1. 常规字符默认是缓存的, 0-9和26个字母大小写

2. 除了常规字符之外都是特殊字符, 都不会缓存, 包括unicode字符

3. all_name_chars中没有长度判断, 所以

.. code-block:: python

    In [1]: x='aaaaaaaaaaaaaaaaaaaa'
    
    In [2]: y='aaaaaaaaaaaaaaaaaaaa'
    
    In [3]: x is y
    Out[3]: True
    
    In [4]: z='aaaaaaaaaaaaaaaaaaaa'
    
    In [5]: x is z
    Out[5]: True
    
    In [6]: x='aaaaaaaaaaaaaaaaaaaab'
    
    In [7]: y='aaaaaaaaaaaaaaaaaaaab'
    
    In [8]: x is y
    Out[8]: True


第一次的x, y, z赋值是20个a, 然后第二次的x, y赋值是20个a加上一个b, 一共21的长度

但是, 为什么:

.. code-block:: python

    In [16]: x='a' * 20
    
    In [17]: y='a' * 20
    
    In [18]: x is y
    Out[18]: True
    
    In [19]: y='a' * 21
    
    In [20]: x='a' * 21
    
    In [21]: x is y
    Out[21]: False

**看下面的长度部分**


PyUnicode_InternInPlace
========================

而PyUnicode_InternInPlace则是处理缓存的具体函数


.. code-block:: c

    void
    PyUnicode_InternInPlace(PyObject **p)
    {
        PyObject *s = *p;
        PyObject *t;
        // 下面是一顿判断s(也就是p)是不是unicode
    #ifdef Py_DEBUG
        assert(s != NULL);
        assert(_PyUnicode_CHECK(s));
    #else
        if (s == NULL || !PyUnicode_Check(s))
            return;
    #endif
        /* If it's a subclass, we don't really know what putting
           it in the interned dict might do. */
        if (!PyUnicode_CheckExact(s))
            return;
        // 这个是判断s是否已经被intern了
        // 判断的依据是PyUnicodeObject->state.interned是否是SSTATE_INTERNED_MORTAL, 也就是1
        if (PyUnicode_CHECK_INTERNED(s))
            return;
        if (interned == NULL) {
            // 这里是初始化interned字典的地方
            interned = PyDict_New();
            if (interned == NULL) {
                PyErr_Clear(); /* Don't leave an exception */
                return;
            }
        }
        // 调用PyDict_SetDefault设置interned字典
        // 返回的是interned中s的值
        //　因为是setdefault操作, 所以如果s已经被赋值过了, 则返回
        // interned中的s的值
        Py_ALLOW_RECURSION
        t = PyDict_SetDefault(interned, s, s);
        Py_END_ALLOW_RECURSION
        if (t == NULL) {
            PyErr_Clear();
            return;
        }
        // -----------注意, 这里是interned中的t不等于s
        // -----------那么把p指针指向的unicode指向interned中的t
        if (t != s) {
            Py_INCREF(t);
            Py_SETREF(*p, t);
            return;
        }
        /* The two references in interned are not counted by refcnt.
           The deallocator will take care of this */
        Py_REFCNT(s) -= 2;
        _PyUnicode_STATE(s).interned = SSTATE_INTERNED_MORTAL;
    }

**缓存的时候, 会指向同一个对象**

foo!这个字符串:

1. 一开始ok_name_char是256空数组
   
2. 然后经过第一个循环之后, ok_name_char赋值了, 比如f这个字符的ascii数值是102, 也就是 *ok_name_char[102] = 1*

3. 然后逐个循环foo!, 循环到!这个字符的时候, 发现!的ascii值是33, 并且ok_name_char[33] == 0, 所以返回0, 不缓存

awd这个字符串

1. 第一个x='awd', ok_name_char返回1, 所以调用PyUnicode_InternInPlace, 缓存了awd

2. 第二个y='awd', ok_name_char返回1, 所以调用PyUnicode_InternInPlace去缓存awd

3. **而PyUnicode_InternInPlace的作用是会把y指向x指向的awd**

4. 所以is操作返回True

那么在函数中呢?

函数中的consts
==================

.. code-block:: python

    In [137]: def test():
         ...:     a = 'foo!'
         ...:     b = 'foo!'
         ...:     print(a is b)
         ...:     return
         ...: 
    
    In [137]: test()
    True

按照之前的说法, foo!不满足缓存条件, 那么a, b调用is操作应该是不同的呀, 为什么会相同呢?


先看看dis的结果

.. code-block:: python

    In [138]: dis.dis(test)
      2           0 LOAD_CONST               1 ('foo!')
                  2 STORE_FAST               0 (a)
    
      3           4 LOAD_CONST               1 ('foo!')
                  6 STORE_FAST               1 (b)
    
      4           8 LOAD_GLOBAL              0 (print)
                 10 LOAD_FAST                0 (a)
                 12 LOAD_FAST                1 (b)
                 14 COMPARE_OP               8 (is)
                 16 CALL_FUNCTION            1
                 18 POP_TOP
    
      5          20 LOAD_CONST               0 (None)
                 22 RETURN_VALUE
    

看到带有LOAD_CONST语句, 并且传入的参数都是1, 看看LOAD_CONST是干嘛的


.. code-block:: c

    // cpython/Python/ceval.c
    PyObject *
    _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
    {
        // 省略了很多代码

        PyObject *consts;

        // 拿到先当前的frame对象
        tstate->frame = f;

        // 拿到co_consts对象
        consts = co->co_consts;

            PREDICTED(LOAD_CONST);
            TARGET(LOAD_CONST) {
                // 从consts拿到对象
                PyObject *value = GETITEM(consts, oparg);
                Py_INCREF(value);
                PUSH(value);
                FAST_DISPATCH();
            }
    }

然后我们看看test函数的codeobject的consts属性

.. code-block:: python

    In [139]: x=test.__code__
    
    In [140]: x
    Out[140]: <code object test at 0x7f1847bebed0, file "<ipython-input-136-22e7d50e9716>", line 1>
    
    In [141]: x.co_consts
    Out[141]: (None, 'foo!')

我们看到, consts包含了一个foo!字符串(虽然我们赋值了两次, 但是只有一个), 然后保存到consts中, 所以当LOAD_CONST执行的时候,

从consts拿到的是同一个foo!(传入的参数都是1)

**所以, 虽然foo!没有被缓存掉(intern), 但是由于codeobject中只存储了一个foo!, 但是LOAD_CONST拿到的是同一个对象, 搜易is返回True**

**这里跟缓存没关系, 只是说函数中拿到的foo!是同一个.**


运行时的缓存(编译优化)
==============================

python会在编译成字节码的时候把常量和常量计算的结果给缓存掉, 带计算的, 运行时计算的结果并不会的, 比如x='a', y='b', c = x + y, c就是运行时计算的.

比如

.. code-block:: python

    In [62]: x='a'
    
    In [63]: y='b'
    
    In [64]: x + y is 'ab'
    Out[64]: False
    
    In [65]: 'a' + 'b' is 'ab'
    Out[65]: True

x+y 和 'a' + 'b'的区别就是, 后一句是编译的时候可以直接执行的

而x + y需要在执行的时候(运行到的时候)再计算, 所以不会intern掉, 而是重新生成x+y的结果, 也就是新的'ab'字符串.

**这里就是编译时候的优化了**

比如在函数中

.. code-block:: python

    In [69]: def test():
        ...:     a = 'foo' + 'bar'
        ...:     return a
        ...: 
    
    In [70]: dis.dis(test)
      2           0 LOAD_CONST               3 ('foobar')
                  2 STORE_FAST               0 (a)
    
      3           4 LOAD_FAST                0 (a)
                  6 RETURN_VALUE


可以看到, 在编译的时候, 已经执行了'foo' + 'bar'的计算结果了


intern手动缓存
====================


sys.intern调用到PyUnicode_InternInPlace, 手动缓存指定的字符

cpython/Python/sysmodule.c

.. code-block:: c

    static PyObject *
    sys_intern(PyObject *self, PyObject *args)
    {
        // s是解析传入参数之后的对象
        PyObject *s;
        if (!PyArg_ParseTuple(args, "U:intern", &s))
            return NULL;
        if (PyUnicode_CheckExact(s)) {
            Py_INCREF(s);
            // 这里调用一下
            PyUnicode_InternInPlace(&s);
            // 然后返回
            return s;
        }
        else {
            PyErr_Format(PyExc_TypeError,
                            "can't intern %.400s", s->ob_type->tp_name);
            return NULL;
        }
    }

字符串的长度和缓存
====================

在all_name_chars那部分中, 针对长度的例子:


.. code-block:: python

    In [1]: x='a' * 20
    
    In [2]: y='a' * 20
    
    In [3]: x is y
    Out[3]: True
    
    In [4]: x='a' * 21
    
    In [5]: y='a' * 21
    
    In [6]: x is y
    Out[6]: False
    
    In [7]: x='aaaaaaaaaaaaaaaaaaaab'
    
    In [8]: y='aaaaaaaaaaaaaaaaaaaab'
    
    In [9]: len(x), len(y)
    Out[9]: (21, 21)
    
    In [10]: x is y
    Out[10]: True

使用\*操作符号, 产生长度21的字符串是不能缓存的, 但是使用常量的21个字符串是可以缓存的, 为什么呢?

看看dis结果:

.. code-block:: c

    In [12]: dis.dis("x='a' * 20")
      1           0 LOAD_CONST               3 ('aaaaaaaaaaaaaaaaaaaa')
                  2 STORE_NAME               0 (x)
                  4 LOAD_CONST               2 (None)
                  6 RETURN_VALUE
    
    In [13]: dis.dis("x='a' * 21")
      1           0 LOAD_CONST               0 ('a')
                  2 LOAD_CONST               1 (21)
                  4 BINARY_MULTIPLY
                  6 STORE_NAME               0 (x)
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE
    
    In [14]: dis.dis("x='aaaaaaaaaaaaaaaaaaaab'")
      1           0 LOAD_CONST               0 ('aaaaaaaaaaaaaaaaaaaab')
                  2 STORE_NAME               0 (x)
                  4 LOAD_CONST               1 (None)
                  6 RETURN_VALUE

看起来, \*重复20以上次数的操作, 不是拿常量了, 而是运行时计算, 所以, x='a'\*21不是编译时候求值, 所以不会走consts缓存的流程




整数的intern
==================

整数也可以intern的, 但是整数的intern应该是和小整数内存池有关.



字符串查找
===============

http://www.laurentluce.com/posts/python-string-objects-implementation/


查找算法参考了BM算法和Horspool算法

