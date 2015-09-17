Python module(2.x)
==================================================

注: 以下都是官方文档 [1]_ 中的摘选

6.1.2 module搜索路径
---------------------
When a module named *spam* is imported, the interpreter first searches for a built-in module with that name. If not found,
it then searches for a file named *spam.py* in a list of directories given by the variable sys.path. sys.path is initialized from these locations:

当一个名为 *spam* 的module被导入的时候,解释器首先会搜寻 **内置的module** , 如果没找到该module,则会根据sys.path中的地址列表搜寻名为 *spam.py* 的文件
(应该是搜寻名为spam的module或者名为spam.py的文件,不仅仅是名为spam.py的文件). 
sys.path是根据下面几个地址内容来初始化的.

* the directory containing the input script (or the current directory).(当前目录)

* PYTHONPATH (a list of directory names, with the same syntax as the shell variable PATH).(环境变量PYTHONPATH)

* the installation-dependent default.(默认安装依赖,也不知道这么解释)


After initialization, Python programs can modify sys.path. The directory containing the script being run is placed at the beginning of the search path,
ahead of the standard library path. This means that scripts in that directory will be loaded instead of modules of the same name in the library directory.
This is an error unless the replacement is intended. See section Standard Modules for more information.

程序可以导入sys, 修改sys.path中第一项为当前路径, 如果在当前路径下存在一个module和其他搜索路径冲突(比如标准库)的话,则会优先导入当前路径的module.

若当前目录有包跟标准库重名的话, 如os, 要导入当前目录的os包,则必须修改sys.path, 把当前目录的路径手动设置在sys.path的第一项,然后reload(sys), 再reload(os)才行.

所以是, 解释器初始化之后, 导入module都是先去找内置的标准库, 找不到才会根据sys.path中的路径以此导包. 但我们可以修改sys.path, 强制先导入当前路径的包.

若当前目录有包跟非标准库重名的话, 如安装了第三方包avatar, 当前路径也有个包叫avatar, 搜索路径就按sys.path来, 也就是说, 当前路径下import avatar, 加载到的是当前路径的avatar.


例如目录结构为

x/
  os/
    __init__.py
    os.py


.. code-block:: python

    In [2]: import os

    In [3]: os
    Out[3]: <module 'os' from '/usr/lib/python2.7/os.pyc'>

    In [4]: import sys

    In [5]: sys.path.insert(0, os.getcwd())

    In [6]: reload(sys)
    <module 'sys' (built-in)>

    In [7]: reload(os)
    <module 'os' from '/path/to/x/os/__init__.pyc'>


6.1.3 "Complied" Python files
-------------------------------

As an important speed-up of the start-up time for short programs that use a lot of standard modules, if a file called spam.pyc exists in the directory where spam.py is found,
this is assumed to contain an already-“byte-compiled” version of the module spam. The modification time of the version of spam.py used to create spam.pyc is recorded in spam.pyc,
and the .pyc file is ignored if these don’t match.

使用内置库会更快,解释器会优先使用.pyc文件. 当.py文件的修改时间和存储在.pyc文件中的修改时间不一致(.pyc中,开头包含了一个4比特的魔术数字(magic number),校验版本,一个4比特的修改时间戳
(timestamp),一个marshalled code object)的时候,pyc文件将被忽略. pyc文件是跨平台的.

Some tips for experts:

* When the Python interpreter is invoked with the -O flag, optimized code is generated and stored in .pyo files. The optimizer currently doesn’t help much; it only removes assert
  statements. When -O is used, all bytecode is optimized; .pyc files are ignored and .py files are compiled to optimized bytecode.

  使用-O选项执行py文件,解释器会优化代码生成.pyo文件, .pyo文件并不会有很大帮助,只是移除了assert语句. 使用了-O选项, 则.pyc文件将会被忽略, py文件将会被编译成.pyo文件.

* Passing two -O flags to the Python interpreter (-OO) will cause the bytecode compiler to perform optimizations that could in some rare cases result in malfunctioning programs.
  Currently only __doc__ strings are removed from the bytecode, resulting in more compact .pyo files. Since some programs may rely on having these available, you should only use
  this option if you know what you’re doing.

  使用-OO选项的话, 也只是讲bytecodezhong 删除__doc__字符, 使得.pyo文件更加紧凑.

* A program doesn’t run any faster when it is read from a .pyc or .pyo file than when it is read from a .py file; the only thing that’s faster about .pyc or .pyo files is the speed
  with which they are loaded.

  .pyc和.pyo文件并不会是得代码执行速度更快了,只是加载更快而已.

* When a script is run by giving its name on the command line, the bytecode for the script is never written to a .pyc or .pyo file. Thus, the startup time of a script may be reduced by
  moving most of its code to a module and having a small bootstrap script that imports that module. It is also possible to name a .pyc or .pyo file directly on the command line.

  使用命令行执行一个文件的时候,并不会生产一个pyc或者pyo文件, 但是可以使用命令行直接执行pyc或者pyo文件

* It is possible to have a file called spam.pyc (or spam.pyo when -O is used) without a file spam.py for the same module. This can be used to distribute a library of Python code in a
  form that is moderately hard to reverse engineer.

  pyc和py文件可以是独立的,pyc可以用来提供给客户端作为分离的lib

* The module compileall can create .pyc files (or .pyo files when -O is used) for all modules in a directory.

  compileall包可以把py文件编译成pyc文件.

Standard Modules: 标准库
-------------------------

Python comes with a library of standard modules, described in a separate document, the Python Library Reference (“Library Reference” hereafter). 
Some modules are built into the interpreter; these provide access to operations that are not part of the core of the language but are nevertheless built in, either for efficiency or
to provide access to operating system primitives such as system calls. The set of such modules is a configuration option which also depends on the underlying platform. For example,
the *winreg* module is only provided on Windows systems. One particular module deserves some attention: sys, which is built into every Python interpreter.

Python包含了标准库(library), 库的描述在PLR文档中. **一些module是内置到解释器(built into interpreter)中的, 这些库提供的操作并不是语言的一部分, 但却是建立在效率或者提供系统调用等操作
系统原语.**


6.4 Package
------------

Contrarily, when using syntax like import item.subitem.subsubitem, each item except for the last must be a package; the last item can be a module or a package but can’t be a class or
function or variable defined in the previous item.

相反地, 使用import item.subitem.subsubitem的形式导入的话, 最后一个item一定要一个package或者module, 不能是类, 函数, 变量.

6.4.1 Importing * From a Package
---------------------------------

Now what happens when the user writes from sound.effects import * ? Ideally, one would hope that this somehow goes out to the filesystem, finds which submodules are present in the
package, and imports them all. This could take a long time and importing sub-modules might have unwanted side-effects that should only happen when the sub-module is explicitly imported.

使用 from sound.effects import * 这样的导入方式, 理想的情况是搜索所有的submodule, 然后一一导入.但是这样可能花费很长的时间并且可能会产生一些不需要的边际效应.
所有最好还是显示指定要导入的module

The only solution is for the package author to provide an explicit index of the package. The import statement uses the following convention: if a package’s __init__.py code defines a
list named __all__, it is taken to be the list of module names that should be imported when from package import * is encountered. It is up to the package author to keep this list
up-to-date when a new version of the package is released. Package authors may also decide not to support it, if they don’t see a use for importing * from their package. For example,
the file sound/effects/__init__.py could contain the following code:

解决方式是显示定义一个package索引.import语句会遵循这样的一个约定: 如果package的__init__.py文件定义了名为__all__的列表, 则from package import * 会导入__all__列表中的module.


.. code-block:: python

    __all__ = ['echo', 'surround', 'reverse']


This would mean that from sound.effects import * would import the three named submodules of the sound package.

If __all__ is not defined, the statement from sound.effects import * does not import all submodules from the package sound.effects into the current namespace; it only ensures that
the package sound.effects has been imported (possibly running any initialization code in __init__.py) and then imports whatever names are defined in the package.
This includes any names defined (and submodules explicitly loaded) by __init__.py. It also includes any submodules of the package that were explicitly loaded by previous import
statements. Consider this code:

如果__all__没有定义, 则from sound.effects import * 则不会将所有的submodule导入到当前命名空间中, 而是只是保证sound.effects已经被导入(可能运行__int__.py中的初始化代码), 之后可以导入
任何在package中定义的submodule, 包括之前使用显示导入的submodule.

.. code-block:: python

    import sound.effects.echo
    import sound.effects.surround
    from sound.effects import *

In this example, the echo and surround modules are imported in the current namespace because they are defined in the sound.effects package when the from...import statement is executed.
(This also works when __all__ is defined.)


Although certain modules are designed to export only names that follow certain patterns when you use import * , it is still considered bad practise in production code.

Remember, there is nothing wrong with using from Package import specific_submodule! In fact, this is the recommended notation unless the importing module needs to use submodules with
the same name from different packages.

这个例子中, 使用from...import语句显示导入了echo和surround, 最后一句的from...import * 只是将sound.effects导入到了当前namespace中, 但是当使用echo和surround的时候,并不会重新加载
这两个module. 后面的就是说from...import * 导入package不好, 推荐使用显示制定导入.

上述导入都是针对导入package的module的, 但对某个module使用from...import * 的时候, 例如当前路径存在一个叫test.py文件, 使用from test import * 则会导入test.py中所有定义的变量,函数,类等等.
在module中也可以定义__all__, 若定义了__all__, 则from test import * 则只会导入__all__中的name.


6.4.2 Intra-package References
-------------------------------

这部分是package中module的绝对/相对导入(absoulte/relative import), 参考PEP-0328 [2]_ 和PEP-0366 [3]_

简单来说, 使用相对导入的时候, 必须把源文件作为module导入, 否则会报 ValueError: Attempted relative import beyond toplevel package

例子:

文件目录结构

m1/
  a.py
   m2/
     b.py
      m3/
        c.py

在c.py中

.. code-block:: python

    # coding=utf-8
    from __future__ import unicode_literals
    from .. import b

    print '__name__: %s' % __name__
    print '__package__' % __package__


from .. import b意味着要从c.py的上一层目录, 也就是m2, 导入b这个module.


将c.py当成module导入, 并且导入的路径层级包含上一层目录, 可以执行

.. code-block:: python

    In [1]: from m1.m2.m3 import c
    __name__: m1.m2.m3.c
    __package__: m1.m2.m3

**__package__中包含了当前目录m3, 以及上一级目录m2, 所以from .. import b可以找到对应的目录, 可以执行**

将c.py当成module导入, 并且导入的路径层级包含上一层目录, 可以执行

.. code-block:: python

    In [2]: import os

    In [3]: os.chdir('m1')

    In [4]: from m2.m3 import c
    __name__: m2.m3.c
    __package__: m2.m3

**__package__中包含了当前目录m3, 以及上一级目录m2, 所以from .. import b可以找到对应的目录, 可以执行**

将c.py当成module导入, 并且从c.py的上一级m2目录导入

.. code-block:: python

    In [7]: os.chdir('m2')

    In [8]: from m3 import c
    ...

    ValueError: Attempted relative import beyond toplevel package

这个时候不可执行, 报错

将c.py修改为

.. code-block:: python

    # coding=utf-8
    from __future__ import unicode_literals

    print '__name__: %s' % __name__
    print '__package__: %s' % __package__

再次从m2目录导入c.py, 有

.. code-block:: python

    In [9]: from m3 import c
    __name__: m3.c
    __package__: m3

__package__中只包含了当前路径, 所以from .. import b中, python找不到..所代表的上一层路径, 运行报错


若直接执行c.py, python m1/m2/m3/c.py, 可得

__name__: __main__
__package__:

或者

python -m python -m m1/m2/m3/c, 有

__name__: __main__
__package__: 


**当使用相对导入的时候,所以必须把源文件当成module导入, 这时因为python对根据__package__来寻找相对包, 找不到就会报ValueError: Attempted relative import beyond toplevel package,
直接执行源文件的话, __package__就为None**


.. [1] https://docs.python.org/2/tutorial/modules.html
.. [2] https://www.python.org/dev/peps/pep-0328/
.. [3] https://www.python.org/dev/peps/pep-0366/
