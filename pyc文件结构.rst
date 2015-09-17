pyc文件结构
==============

pyc文件是python解释器将py文件编译成的一个二进制文件. 虽然一般说法是, python是一个 *解释* 语言, 但实际上存在编译过程,所以也不能简单地把python说成是解释语言 [1]_ .

python语言定义了一些语法格式标准, 然后python解释器会把py文件编译成一个pyc文件, pyc文件是一个二进制的字节码(bytecode [2]_ )文件, 里面的格式是python指定的,
然后python的解释器就执行字节码了.

一般说的python都是是CPython [3]_, 一个标准的C编写的python实现(implementation), 可以与C语言集成.

python的实现还包括Jython, 将py文件编译成java的字节码, 可以在JVM中执行, 也可以与java集成, IronPython, Microsoft平台下的实现 [4]_.

JVM和python的解释器相比, 比较高效, 部分原因还是因为JIT [5]_. python中也有很多JIT的实现, 包括PyPy, Pyston [6]_ 等


注: 以下部分都是参考 [7]_ 中的摘选

At the simple level, a .pyc file is a binary file containing only three things:

    * A four-byte magic number,
    * A four-byte modification timestamp, and
    * A marshalled **code object**.

简单来说, pyc文件是二进制文件, 包含了一下内容:

    * 4比特的魔术数字
    * 4比特的修改时间戳
    * 一个打包(不太理解marshalled)好的 **code_object**.


The magic number is nothing as cool as `cafebabe <http://www.artima.com/insidejvm/whyCAFEBABE.html>`_, it's simply two bytes that change with each change to the marshalling code, and then two bytes of 0d0a. 
The 0d0a bytes are a carriage return and line feed, so that if a .pyc file is processed as text, it will change, and the magic number will be corrupted. This will keep the file from
executing after a copy corruption. The marshalling code is tweaked in every major release of Python, so in practice the magic number is unique in each version of the Python
interpreter. For Python 2.5, it's b3f20d0a. (the gory details are in import.c)

魔术数字与java类文件中的魔术数字: cafebabe_ 一样, 起到标识作用. 魔术数字的前两个比特会随着解释器的不同而不同(原文中的marshalling code应该是把用户源码编译为pyc文件的代码, 应该就是解释器,
并且前两个比特在经过解析之后, 就是 `import.c <http://svn.python.org/view/python/trunk/Python/import.c?view=markup>`_ 中的每个解释器特有的数字), 接着两比特是0d0a. 0d0a就是换行和回车符.
如果一个pyc文件被处理成text文件, 则魔术数字会被破坏(腐化), 这样可以避免执行一个已被破坏的pyc文件.

例如, 一个test.pyc文件, 使用python2.7.5导入之后生成的test.pyc文件

.. code-block:: python

    In [55]: f=open('test.pyc', 'rb')

    In [56]: magic=f.read(4)

    In [57]: magic
    Out[57]: '\x03\xf3\r\n'

    In [58]: struct.unpack('<H', magic[:2])
    Out[58]: (62211,)

    In [59]: magic.encode('hex')
    Out[59]: '03f30d0a'


The four-byte modification timestamp is the Unix modification timestamp of the source file that generated the .pyc, so that it can be recompiled if the source changes.

接着的4比特是unix风格的修改的时间戳. 当导入一个module的时候, 解释器会对比源文件修改时间和pyc文件中这个修改时间戳, 不一致就会重新编译.

The entire rest of the file is just the output of `marshal.dump <https://docs.python.org/2/library/marshal.html>`_ of the code object that results from compiling the source file.
Marshal is like pickle, in that it serializes Python objects. It has different goals than pickle, though. Where pickle is meant to produce version-independent serialization suitable
for persistence, marshal is meant for short-lived serialized objects, so its representation can change with each Python version. Also, pickle is designed to work properly for
user-defined types, while marshal handles the complexities of Python internal types. The one we care about in particular here is the code object.

之后pyc所有的内容就是调用marshal.dump之后输出的内容. Marshal模块类似与picke, 也是为了序列化python对象. 但是两者目的不一样. picke序列的结果是独立于版本的持久化对象. 而
marshal的结果在每一个python版本下都可能不一样的, marshal操作的是python内置类型.

Marshal模块文档中的介绍:

This module contains functions that can read and write Python values in a binary format. The format is specific to Python, but independent of machine architecture issues
(e.g., you can write a Python value to a file on a PC, transport the file to a Sun, and read it back there). Details of the format are undocumented on purpose; it may change between
Python versions (although it rarely does).

简而言之: Marshal是用python指定的二进制格式读写python的内容(变量,常量,类等等), 独立与平台但是依赖于解释器版本. 也就是读写python字节码, 读取的时候会组成一个code object对象.

.. code-block:: python

    In [20]: x=marshal.load(pycf)

    In [21]: x
    Out[21]: <code object <module> at 0x7f16bebe4b30, file "test.py", line 3>

    In [22]: marshal.dumps('a=1')
    Out[22]: 's\x03\x00\x00\x00a=1'

    In [23]: marshal.dumps(1)
    Out[23]: 'i\x01\x00\x00\x00'

而python字节码怎么执行, 需要 `dis模块 <https://docs.python.org/2/library/dis.html>`_

.. code-block:: python

    In [25]: dis.disassemble(x)
      3           0 LOAD_CONST               0 (0)
                  3 STORE_NAME               0 (a)

      4           6 LOAD_NAME                0 (a)
                  9 POP_JUMP_IF_TRUE        20

      5          12 LOAD_CONST               1 ('not a')
                 15 PRINT_ITEM          
                 16 PRINT_NEWLINE       
                 17 JUMP_FORWARD             5 (to 25)

      7     >>   20 LOAD_CONST               2 ('hell, it is A a')
                 23 PRINT_ITEM          
                 24 PRINT_NEWLINE
            >>   25 LOAD_CONST               3 (None)
                 28 RETURN_VALUE        

    In [26]: def test():
       ....:     print 'test function'
       ....:

    In [28]: dis.dis(test)
      2           0 LOAD_CONST               1 ('test function')
                  3 PRINT_ITEM          
                  4 PRINT_NEWLINE       
                  5 LOAD_CONST               0 (None)
                  8 RETURN_VALUE        



.. [1] Python is an interpreted language ?: http://stackoverflow.com/questions/2998215/if-python-is-interpreted-what-are-pyc-files
.. [2] Bytecode: https://en.wikipedia.org/wiki/Bytecode
.. [3] PEP-0339(Design of the CPython Compiler): https://www.python.org/dev/peps/pep-0339/
.. [4] Python的多个实现, Jython, Ironython, PYPY: http://www.toptal.com/python/why-are-there-so-many-pythons
.. [5] JIT: https://en.wikipedia.org/wiki/Just-in-Time_Manufacturing
.. [6] 旨在在Python内置JIT的项目Pyston的一篇blog: https://blogs.dropbox.com/tech/2014/04/introducing-pyston-an-upcoming-jit-based-python-implementation/
.. [7] pyc文件结构: http://nedbatchelder.com/blog/200804/the_structure_of_pyc_files.html
.. [8] protecting a python codebase: http://bits.citrusbyte.com/protecting-a-python-codebase/

