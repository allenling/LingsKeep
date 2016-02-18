Python 2.7.x
=================

read xlrd datetime
-------------------

datetime_tuple = xlrd.xldate_as_tuple(cell_value, book.datemode)

datetime_tuple的格式为(year, month, day, hour, min, sec)


windows下, 使用ConfigParser读取ini文件编码问题
-----------------------------------------------

windows下, 使用ConfigParser读取ini文件需要指定编码为: utf_8_sig

Use utf_8_sig charset and open your file using io.open() or codecs.open() to get unicode sections, options and values.

.. code-block:: python

    from ConfigParser import ConfigParser
    import io

    # create an utf-8 .ini file with a BOM marker
    with open('bla.ini', 'wb') as fp:
        fp.write(u'[section]\ncl\xe9=value'.encode('utf_8_sig'))

    # read the .ini file
    config = ConfigParser()
    with io.open('bla.ini', 'r', encoding='utf_8_sig') as fp:
        config.readfp(fp)

    # dump the config
    for section in config.sections():
        print "[", repr(section), "]"
        for option in config.options(section):
            value = config.get(section, option)
            print "%r=%r" % (option, value)

参考: https://bugs.python.org/issue7519

read and write a file
-----------------------
读写文件注意的是read完之后, write之前, 把文件内容指针归0, 最后truncate
如在一个目录下所有的文件的头部加上

# coding=utf-8

from __future__ import unicode_literals

.. code-block:: python

    import os
    for root, dirs, files in os.walk(target_dir):
        for f in files:
            full_path = os.path.join(root, f)
            with open(full_path, 'r+') as t:
                content = t.read()
                content = '# coding=utf-8\nfrom __future__ import unicode_literals\n' + content
                t.seek(0)
                t.write(content)
                t.truncate()

context management
------------------------

with 语句在PEP343中有论述, 一般使用就是with的开头调用__enter__方法, 最后调用__exit__方法, 可以看成是try...finally语句.

.. code-block:: python

    with EXPR as VAR:
        BLOCK

相等于

.. code-block:: python

    mgr = (EXPR)
    exit = type(mgr).__exit__  # Not calling it yet
    value = type(mgr).__enter__(mgr)
    exc = True
    try:
        try:
            VAR = value  # Only if "as VAR" is present
            BLOCK
        except:
            # The exceptional case is handled here
            # PEP343中是这样，其实感觉应该是这样
            # if exit(mgr, *sys.exc_info()):
            #     raise
            # 因为exit返回False的时候，是不会抛出异常的
            exc = False
            if not exit(mgr, *sys.exc_info()):
                raise
            # The exception is swallowed if exit() returns true
    finally:
        # The normal and non-local-goto cases are handled here
        if exc:
            exit(mgr, None, None, None)

注意的是
1. __exit__方法不建议re-raise异常，应该是上层去处理异常
2. 希望__exit__中不抛出(指定)异常，可以返回False, 返回True则是正常抛出异常. if arg[0] is Keyerror: return False
3. contextlib.contextmanager指支持yield一次的生成器,　因为contextlib.contextmanager只在__enter__和__exit__方法各调用生成器的next方法一次，若生成器还未终止，引发异常.


