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


Generator and send
-----------------------

yield　返回值并挂起, 生成器产生斐波拉契数列

.. code-block:: python

    import itertool


    def F():
        a,b = 0,1
        yield a
        yield b
        while True:
            a, b = b, a + b
            yield b

    # 生成器实例
    f = F()

    # 迭代10次
    list(itertools.islice(f, 10))

yield 关键字既可以返回值给调用函数，也可以 **接收** 调用函数穿进来的数值

.. code-block:: python

    def coroutine():
        for i in range(1, 5):
            x = yield i
            print("From Generator {}".format((x)))
    c = coroutine()
    c.send(None)
    try:
        while True:
            print("From user {}".format(c.send(1)))
    except StopIteration:
        pass

输出为

From Generator 1

From user 2

From Generator 1

From user 3

From Generator 1

From user 4

From Generator 1

执行流程为

**第一次迭代生成器的时候，必须使用send(None)或者next()，不能send一个非None值**

所以一开始，c.send(None)，启动生成器, 生成器执行到yield i的时候，返回1, 挂起，之后c.send(1), 则x等于send进来的值，为1，然后生成器继续执行，直到下一个yield, 这时打印出From Generator 1(x的值), 然后i=2, 再次执行到yield i, 返回i, 然后挂起．而上层语句或得返回值2，然后打印From user 2，然后继续send, 一次重复

读取一个py/pyc文件的变量, 使用内置函数execfile.
------------------------------------------------

..code-block:: python

    cfg = {}
    execfile(my_file, cfg, cfg)
    execfile会把除cfg中的变量赋值到cfg中.
    # 一般解析之后, cfg['__builtins__']会包含__builtins__的很多很多变量
    cfg = {'__builtins__': {...}}
    # 为了方便, 可以这样
    cfg = {'__builtins__': __builtins__}把__builtins__排除在外.

__name__和__main__
----------------------

在通常情况下, 若我们直接执行py文件, 在文件最后, 会写上

.. code-block:: python

  def main():
      pass

  if __name__ == '__main__':
      main()

但是__name__ is '__main__'为False. is是比对对象的identity, 在python中就是id返回的值, 也就是内存地址. ==只是比对值, 每个对象都可以有一个__eq__的方法, 当使用==的时候, 就是调用
该方法.

所以, __main__是一个常量, 而__name__是解释器在执行py文件(或者导入module)的时候赋值为'__main__', 两者id出来是不一样的.

.. code-block:: python

   >>> a = 'pub'
   >>> b = ''.join(['p', 'u', 'b'])
   >>> a == b
   True
   >>> a is b
   False

file.flush
-------------

文档中这么说明的:

flush() does not necessarily write the file’s data to disk. Use flush() followed by os.fsync() to ensure this behavior.

所以, 调用os.flush并不意味着会写数据到磁盘, 而是只是说数据已经从程序的维护的内存buffer, 被复制操作系统的内存buffer中, 在下一个操作系统的fsync操作会被写入到磁盘.

http://stackoverflow.com/questions/7127075/what-exactly-the-pythons-file-flush-is-doing

1. The first, flush, will simply write out any data that lingers in a program buffer to the actual file. Typically this means that the data will be copied from the program buffer to the operating system buffer.

2. Specifically what this means is that if another process has that same file open for reading, it will be able to access the data you just flushed to the file. However, it does not necessarily mean it has been "permanently" stored on disk.

3. To do that, you need to call the os.fsync method which ensures all operating system buffers are synchronized with the storage devices they're for, in other words, that method will copy data from the operating system buffers to the disk.

**flush socket**

在python的关于socket的文档中, 出现了关于flush socket的说明

Now there are two sets of verbs to use for communication. You can use send and recv, or you can transform your client socket into a file-like beast and use read and write. The latter is the way Java presents its sockets. I’m not going to talk about it here, except to warn you that you need to use flush on sockets. These are buffered “files”, and a common mistake is to write something, and then read for a reply. Without a flush in there, you may wait forever for the reply, because the request may still be in your output buffer.

其实是说, 若你把socket包装成一个file-like对象的话, 注意要flush而已. file-like对象中, write只是写入到程序维护buffer而已, flush才能把数据发送到操作系统维护的buffer, 就socket来说

, flush才是send的效果. 如果只是使用send方法, 就没有flush的必要了.

字符串打印
----------

一个例子:

template = '''<iron-meta type="mainMenus" key="name" value=%s ></iron-meta>''', value后面跟一个js的列表.

template中的value的值是一个list, 在polymer中, 若要在html中传入对象(如字典, 列表这些非字符串), 则必须是单引号开始, 接着对象开始符(字典就是{, 列表就是[), 然后里面的字符串都是用双引号来包含, 接着
对象结束符(字典就是}, 列表是]), 然后一个单引号.

所以template应该是这样的形式:

若value=['v11', 'v12']

<iron-meta type="mainMenus" key="name" value='["v11", "v12"]' ></iron-meta>


  1. __str__和__repr__进行格式化
  
  
     我们有:
     str(value).__str__() == "['v11', 'v12']"
     所以想法是直接单引号换成双引号
  
     .. code-block:: python
  
        In [1]: template % (str(value).replace("'", '"'))
        Out[1]: <iron-meta type="mainMenus" key="myname" value=["v11", "v12"] ></iron-meta>
  
     我们发现单引号不见了. 对于str(value), 我们有
  
     .. code-block:: python
  
        In [2]: str(value)
        Out[2]: "['v11', 'v12']"
  
        In [3]: str(value)[0]
        Out[3]: '['
  
        In [4]: print str(value)[0]
        Out[4]: ['v11', 'v12']
     
     所以, str(value)的第一个字符就是[, 但是打印出来的时候多了一个", 所以"应该表示字符串开始.
     然后, 既然我们是用来__str__, 那么我们可以试试__repr__
  
  
     .. code-block:: python
  
        In [5]: str(value).__repr__()
        Out[5]: '"[\'v11\', \'v12\']"'
  
     带有转义符的字符串, 我们使用print打印出来
  
     .. code-block:: python
  
        In [6]: print str(value).__repr__()
        Out[6]: "['v11', 'v12']"
  
        In [7]: str(value).__repr__()[0]
        Out[7]: '"'
     
      __repr__输出的结果中, 第一个字符就是", 而不是[.
      
      而__str__返回的结果是"['v11', 'v12']", 但是第一个字符是[, 所以打印__str__的第一个的, 开始的双引号表示字符串的开始.
      而__repr__返回的结果中, 第一个字符是单引号, 表示字符串开始, 然后接着双引号, 双引号才是第一个字符, 打印__repr__的第一个的, 开始的双引号其实是字符串的一部分.
      也就是说__repr__的结果就是原生的字符串. 所以我们可以把单引号转成双引号, 双引号转成单引号就行
  
     .. code-block:: python
  
        In [8]: str(value).replace("'", '"').__repr__()
        Out[8]: '\'["v11", "v12"]\''
  
        In [9]: print str(value).replace("'", '"').__repr__()
        Out[9]: '["v11", "v12"]'
   
     这里, 就可以满足要求了.
  
     .. code-block:: python
  
        In [10]: template % str(value).replace("'", '"').__repr__()
        Out[10]: '<iron-meta type="mainMenus" key="name" value=\'["v11", "v12"]\' ></iron-meta>'
  
        In [11]: print template % str(value).replace("'", '"').__repr__()
        Out[11]: <iron-meta type="mainMenus" key="name" value='["v11", "v12"]' ></iron-meta>
     
     然后直接复制到html文件去就好了

  2. StringIO

  基本上StringIO.StringIO就是逐个字符串逐个字符串写进去就好, 让他们帮你格式化好的.

  对于上面的value, 我们希望得到的字符串形式是第一个是单引号, 然后是列表开始符, 然后列表每个每个元素都用双引号包含, 然后列表结束符, 然后单引号的形式,
  所以, 我们只要逐个字符写到StringIO去就好了.

  .. code-block:: python

     In [1]: container = StringIO.StringIO()

     In [2]: container.write("'")

     In [3]: for i in value:
                 # 这里简单点就都转成双引号
                 tmpi = i.replace("'", '"')
                 container.write(tmpi)
     
     In [4]: container.write("'")

     In [5]: container..getvalue()
     Out[5]: '\'["v11", "v12"]\''

     In [6]: print container.getvalue()
     Out[6]: '["v11", "v12"]'

     In [7]: template % (container.getvalue())
     Out[7]: '<iron-meta type="mainMenus" key="name" value=\'["v11", "v12"]\' ></iron-meta>'

     In [8]: print template % (container.getvalue())
     Out[8]: <iron-meta type="mainMenus" key="name" value='["v11", "v12"]' ></iron-meta>















