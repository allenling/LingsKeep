read xlrd datetime
===================

datetime_tuple = xlrd.xldate_as_tuple(cell_value, book.datemode)

datetime_tuple的格式为(year, month, day, hour, min, sec)


windows下, 使用ConfigParser读取ini文件编码问题
===============================================

**python2.7**

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
=======================

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
========================

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
=======================

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
================================================

..code-block:: python

    cfg = {}
    execfile(my_file, cfg, cfg)
    execfile会把除cfg中的变量赋值到cfg中.
    # 一般解析之后, cfg['__builtins__']会包含__builtins__的很多很多变量
    cfg = {'__builtins__': {...}}
    # 为了方便, 可以这样
    cfg = {'__builtins__': __builtins__}把__builtins__排除在外.

__name__和__main__
======================

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
=============

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
==========

一个例子:

template = '''<iron-meta type="mainMenus" key="name" value=%s ></iron-meta>''', value后面跟一个js的列表.

template中的value的值是一个list, 在polymer中, 若要在html中传入对象(如字典, 列表这些非字符串), 则必须是单引号开始, 接着对象开始符(字典就是{, 列表就是[), 然后里面的字符串都是用双引号来包含, 接着
对象结束符(字典就是}, 列表是]), 然后一个单引号.

所以template应该是这样的形式:

若value=['v11', 'v12']

<iron-meta type="mainMenus" key="name" value='["v11", "v12"]' ></iron-meta>


1. __str__和__repr__进行格式化
---------------------------------

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
----------------------

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


变量赋值
=========

python的变量赋值其实是一个名字指向一个值, 也就是所谓的引用.

比如a=1, 将a这个名字指向1, 然后a=1, 也是将a这个名字指向1, 在python中, 这个时候是复用了常量1, 所以id(x) == id(a)

然后x=1, a=x的情况是, 名字x指向1, 然后a指向x所指向的值, 也就是1, 所以还是id(x) == id(a), 若x=2, 这个时候x就指向另外一个常量2, 所以x为2, a还是为1, 这个时候id(x)并不等于id(a)

对于字符串和数字之类的不可变对象(包括tuple), 修改会rebind, reassign一个新的值, 比如

.. code-block:: python

    x='a'
    v1 = id(x)
    x += 'b'
    v2 = id(x)

这个时候v1和v2并不像等, 也就是其实是x这个名字被指向一个新的字符串了. 同理, x=1, x+=1的情况也一样.

而对于可变对象, 也就是支持modify in place, 也就是可修改的对象, 修改并不会rebind, reassign, 是在原处修改, 除非手动rebind, reassign, 否则指向的值(也就是可变对象)不变.

如

.. code-block:: python

    x = [1]
    v1 = id(x)
    x.append(2)
    v2 = id(x)

v1和v2是相等的, 只是原处修改了值, 名字x指向的值没有变.

对于

.. code-block:: python

    x = [1]
    a=x
    x.append(2)

这种情况下, a先指向了x所指向的值, 是一个可变对象, 然后修改了该可变对象, 所以a的值也是[1,2], 所以id(a)等于id(x)

然后对x重新赋值, 也就是rebind, reassign

x=[1]

这个时候x指向一个新的值, 但是a指向并没有变化, 所以id(a)并不等于id(x)


所以在python中, 赋值都是指向一个值, 值分为可变和不可变对象, 可变对象支持modify in place, 然后一旦修改不可变对象, 其实是重新rebind, reassign, 也就是重新
指向一个新的值, 而修改可变对象的话, 是原处修改, 所以指向的值仍然没有变化.

这里可以拓展到使用可变对象作为函数默认值的情况, 由于python的函数默认值是在函数定义的时候就赋值了, 而不是每次执行的时候赋值(Python’s default arguments are evaluated once when the function is defined, not each time the function is called, http://docs.python-guide.org/en/latest/writing/gotchas/).

其实可以这么理解, 一旦函数定义了, 函数对象自己就将默认值存起来, 存在func.func_defaults, 变量的名称存在func.func_code.co_varnames, 然后每次调用就记住了.

例如:

.. code-block:: python

    def test(a, b=[]):
        b.append(1)
        print a, b

调用和输出:

.. code-block:: 

    test(1)
    1, [1]
    test(1)
    1, [1, 1]
    test(1)
    1, [1, 1, 1]

一开始,

test.func_defaults为([],)

test.func_code.co_varnames为('a', 'b')

调用第一次之后, 

test.func_defaults为([1],)

调用第二次之后, 

test.func_defaults为([1, 1],)

以此类推

若默认值不是可变对象, 则存在func.func_defaults的值是不会变的

.. code-block:: python

    def test(a, b=1):
        b += 1
        print a, b

调用和输出:

.. code-block:: 

    test(1)
    1, 2
    test(1)
    1, 2
    test(1)
    1, 2

无论调用几次, test.func_defaults都是(1)

这是因为func.func_defaults是一个元组，不能改变元组的值，但是你可以改变元组里面可变对象的值.

x = ('a', [])

x[1].append('data')

x == ('a', ['data'])

关于tuple的一个小"问题"

.. code-block:: python

    In [27]: x = ([], 'a')
    
    In [28]: x[0].append('b')
    
    In [29]: x
    Out[29]: (['b'], 'a')
    
    In [30]: x[0] += ['c']
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-30-131b014e09ec> in <module>()
    ----> 1 x[0] += ['c']
    
    TypeError: 'tuple' object does not support item assignment
    
    In [31]: x
    Out[31]: (['b', 'c'], 'a')
    

tuple不可更改, 但是上面例子中的列表却可以更改，并且报了异常之后也能修改成功, 这是因为可变对象是in-place修改的原因.

is或者==
============

https://docs.python.org/2/reference/datamodel.html

Every object has an identity, a type and a value. An object’s identity never changes once it has been created; you may think of it as the object’s address in memory. The ‘is‘ operator compares the identity of two objects; the id() function returns an integer representing its identity (currently implemented as its address).

python中一切都是对象, is是比较对象内存地址, ==是比较值的. id出来的就是对象的值. 

对于不可变对象, 任何rebind, reassign都是创建一个新对象, 所以

y = 'abc'

x = 'a' + 'b' + 'c'

a = ''.join(['a', 'b', 'c'])

x == y == a

x is y为True(可能是python内部把值, 也就是abc字符串, 复用了)

x is a 为False


另外, 关键字True和False的id是一定的, 在python2.7.6中True的id为9544944, False的id为9544496.

但是在python2.x中, True和False都不是关键字, 所以可以随意赋值, python3.x中这两者都是关键字了.


python"定时任务"
=================

一般是100个字节码指令之后去检查信号或者切换线程等.

https://docs.python.org/2/library/sys.html#sys.setcheckinterval

sys.setcheckinterval(interval)

Set the interpreter’s “check interval”. This integer value determines how often the interpreter checks for periodic things such as thread switches and signal handlers.

The default is 100, meaning the check is performed every 100 Python virtual instructions. Setting it to a larger value may increase performance for programs using threads.

Setting it to a value <= 0 checks every virtual instruction, maximizing responsiveness as well as overhead.

**python3.2之后的gil是不是基于100字节然后切换的形式了, 是基于时间的**



global
--------

例子:

.. code-block:: python

    import dis
    
    d = {'a': 1}
    
    x = True
    
    
    def test():
        print x
        print d
    
    
    def test1():
        print x
        x = False
        print x
    
    
    def test_mutable():
        print d
        d['a'] = 2
        print d
    
    test_mutable()
    
    dis.dis(test)
    print 'test'
    dis.dis(test1)
    print 'test1'
    dis.dis(test_mutable)

输出

.. code-block:: 

    {u'a': 1}
    {u'a': 2}
    
    12           0 LOAD_GLOBAL              0 (x)
                 3 PRINT_ITEM          
                 4 PRINT_NEWLINE       
    
    13           5 LOAD_GLOBAL              1 (d)
                 8 PRINT_ITEM          
                 9 PRINT_NEWLINE       
                10 LOAD_CONST               0 (None)
                13 RETURN_VALUE        
    
    test
    
    17           0 LOAD_FAST                0 (x)
                 3 PRINT_ITEM          
                 4 PRINT_NEWLINE       
    
    18           5 LOAD_GLOBAL              0 (False)
                 8 STORE_FAST               0 (x)
    
    19          11 LOAD_FAST                0 (x)
                14 PRINT_ITEM          
                15 PRINT_NEWLINE       
                16 LOAD_CONST               0 (None)
                19 RETURN_VALUE        
    
    test1
    
    23           0 LOAD_GLOBAL              0 (d)
                 3 PRINT_ITEM          
                 4 PRINT_NEWLINE       
    
    24           5 LOAD_CONST               1 (2)
                 8 LOAD_GLOBAL              0 (d)
                11 LOAD_CONST               2 (u'a')
                14 STORE_SUBSCR        
    
    25          15 LOAD_GLOBAL              0 (d)
                18 PRINT_ITEM          
                19 PRINT_NEWLINE       
                20 LOAD_CONST               0 (None)
                23 RETURN_VALUE        

上面test中, 字节码是LOAD_GLOBAL, 这是加载全局变量, 而test1是LOAD_FAST, 加载局部变量, 为什么有区别?

https://docs.python.org/2/faq/programming.html#what-are-the-rules-for-local-and-global-variables-in-python

In Python, variables that are only referenced inside a function are implicitly global. If a variable is assigned a value anywhere within the function’s body, it’s assumed to be a local unless explicitly declared as global.

Though a bit surprising at first, a moment’s consideration explains this. On one hand, requiring global for assigned variables provides a bar against unintended side-effects.

On the other hand, if global was required for all global references, you’d be using global all the time. You’d have to declare as global every reference to a built-in function or to a component of an imported module.

This clutter would defeat the usefulness of the global declaration for identifying side-effects.

关键在于上面句子中的 If a variable is assigned a value anywhere within the function’s body, it’s **assumed** to be a local unless explicitly declared as global.这句话中的If a variable is assigned, 是assigned的时候

才会当(assumed)成局部变量.

所以可以这么理解, 在函数里面, 修改字典的时候, 是modify in place, 不是reassign, rebinding, 所以解释器会直接根据LEGB原则加载到全局的字典, 然后修改.

而其他不可变对象就不行, 不可变对象在函数里面若有修改操作, 也就是reassign, rebinding操作, 解释器就把它当成局部变量, 因为test中只有打印语句就报没有定义的错误, 而test1中有reassign, 则

解释器就把x当做局部遍历, 当你要操作一个局部变量的时候，必须先赋值

有时候得抠字眼一下才能理解.

先赋值
-------------

.. code-block:: python

    def test():
        print(x)
    
    x = 1
    
    test()

看起来print(x)是在x赋值之前就调用了，应该报未赋值的异常，但是其实是可以正常运行的

这是因为先赋值是指 **操作之前** 必须赋值, 当解释器执行到test函数定义的时候，是执行了函数定义操作，但是并没有操作x, 然后我们赋值x, 然后调用test, test中操作了x

所以这个情况就是操作之前赋值了

如果把x的赋值移到test调用之后, 就会报未赋值异常了

一个gotcha情况
-------------------

下面的代码运行会是什么情况(对象的__iadd__一般返回self)

.. code-block:: python

    In [1]: x=(1,2,[1])
    
    In [2]: x[2] += [2,3]

http://www.dongwm.com/archives/Python%E5%85%83%E7%BB%84%E7%9A%84%E8%B5%8B%E5%80%BC%E8%B0%9C%E9%A2%98/

但是
    
.. code-block:: python

    In [1]: x=(1, 2, [1])

    In [2]: a=x[2]
    
    In [2]: a += [2, 3]

这样是不报错并且修改成功，所以对比起来就知道了，x[2] += [2,3]在报错在于x[2]的赋值，由于元组不能重新赋值才会报错，但是你拿到元祖中可变对象，然后修改可变对象，就是可以的了

python3 Design and History FAQ
==================================


https://docs.python.org/3/faq/design.html


sorted(list.sort)
====================

**python3**

1. key函数: 取消了cmp而使用key, 在python2.x之前, cmp会每遍历一个元素就调用一次, 而指定key函数之后, key函数只运行一次，然后根据key返回的值列表排序，很多数据的时候比起cmp, 性能会更好点
   
   https://www.zhihu.com/question/30389643 

   The value of the key parameter should be a function that takes a single argument and returns a key to use for sorting purposes. This technique is fast because the key function is called exactly once for each input record.
   (https://docs.python.org/3/howto/sorting.html)

2. timsort算法: The Timsort algorithm used in Python does multiple sorts efficiently because it can take advantage of any ordering already present in a dataset.

   https://bindog.github.io/blog/2015/03/30/use-formal-method-to-find-the-bug-in-timsort-and-lunar-rover/
   
   timsort的一个安全问题: http://www.envisage-project.eu/proving-android-java-and-python-sorting-algorithm-is-broken-and-how-to-fix-it/

3. key中使用operation.itemgetter/attrgetter/methodcaller更快/更好/更清楚, why?

1. python基本类型的比较都是C级别的, 而对象级别的(__eq__等方法)都是python基本的

2. The operator module in python implements many functions of common use in C, making them faster.(http://bioinfoblog.it/2010/01/operator-itemgetter-rocks/), 但是有个operator.py, 你说是C实现的，表示怀疑

3. 有些人说用itemgetter比起lamba, 能更清楚表示在干嘛

4. 好吧，网上都说operator里面的函数比较快，但是为什么快呢，有些人说是C实现的，但是itemgetter的实现明显是python代码, 而且官方文档也没说明，就一句
   The key-function patterns shown above are very common, so Python provides convenience functions to make accessor functions easier and faster. The operator module has itemgetter(), attrgetter(), and a methodcaller() function.
   (https://docs.python.org/3/howto/sorting.html)

浮点数运算
=============

直接的浮点数运算一定会有偏差值的, 因为依赖于底层, 如果想用精确值, 那么建议用decimal模块

https://docs.python.org/3/faq/design.html#why-are-floating-point-calculations-so-inaccurate

http://python3-cookbook.readthedocs.io/zh_CN/latest/c03/p02_accurate_decimal_calculations.html

.. code-block:: python

    >>> a = 4.2
    >>> b = 2.1
    >>> a + b
    6.300000000000001
    >>> (a + b) == 6.3
    False
    >>>



python interpreter(vm)
========================

https://www.ics.uci.edu/~brgallar/week9_3.html

vs java vm: https://stackoverflow.com/questions/441824/java-virtual-machine-vs-python-interpreter-parlance

GIL
---------

C拓展的代码是不会释放GIL的, 可以手动释放: https://stackoverflow.com/questions/651048/concurrency-are-python-extensions-written-in-c-c-affected-by-the-global-inter

GIL的C接口: https://docs.python.org/3/c-api/init.html#thread-state-and-the-global-interpreter-lock



python gc
============

http://www.wklken.me/posts/2015/09/29/python-source-gc.html

https://docs.python.org/3/faq/design.html#how-does-python-manage-memory

The standard implementation of Python, CPython, uses reference counting to detect inaccessible objects, and another mechanism to collect reference cycles, periodically

executing a cycle detection algorithm which looks for inaccessible cycles and deletes the objects involved

CPython是引用计数为主, 当引用计数为0的时候, 触发回收(这个回收是不是真正的回收, 看情况, 有可能只是放到内存池里面)

如果确定程序没有循环引用的情况或者不在乎, 那么可以关掉gc, gc.disbale, 即使禁用了gc, 引用计数也是可以正常工作的, 因为引用计数是数字减少到0的时候触发, 而不是周期性的运行.

减少副本的引用计数实现的:

http://www.wklken.me/posts/2015/09/29/python-source-gc.html

检测:

每个容器对象创建的时候会加入到0代链表, 然后如果容器引用计数减少为0, 把容器对象从0代链表中移除, 触发引用计数回收, 如果引用计数没有减少到0, 继续待着

然后0代链表上对象数量打到阀值, gc开始, stop the world, 拷贝一份0代链表的对象的副本, 然后遍历一下(遍历函数是各个容器对象自己有定义的, 比如字典就是遍历每一个key和value的对象),

引用计数减少1(比如字典, 就是key和value的对象的引用计数都减少1)

然后开始标记:

那些引用计数为0的对象称为不可达对象, 其他引用计数大于0的称为可达对象,移动到1代链表，继续

然后回收不可达对象


循环引用的问题和__del__
--------------------------

https://www.holger-peters.de/an-interesting-fact-about-the-python-garbage-collector.html

如果定义了__del__方法, 那么循环引用则不会被gc掉

python2.7中

.. code-block:: python

    In [1]: import gc
    
    In [2]: class T:
       ...:     def __del__(self):
       ...:         print('in T __del__')
       ...:         
    
    In [3]: a=T()
    
    In [4]: b=T()
    
    In [5]: a.other = b
    
    In [6]: b.other = a
    
    In [7]: a=None
    
    In [8]: b=None
    
    In [9]: gc.collect()
    Out[9]: 67
    
    In [10]: gc.garbage
    Out[10]: 
    [<__main__.T instance at 0x7f80798a3518>,
     <__main__.T instance at 0x7f80798ff8c0>]

gc不能回收a和b之前指向的对象, 因为__del__方法被定义了, 如果没有定义__del__方法呢

.. code-block:: python

    In [1]: import gc
    
    In [2]: class T:
       ...:     pass
       ...: 
    
    In [3]: a=T()
    
    In [4]: b=None
    
    In [5]: b=T()
    
    In [6]: a.other = b
    
    In [7]: b.other = a
    
    In [8]: a=None
    
    In [9]: b=None
    
    In [10]: gc.collect()
    Out[10]: 53
    
    In [11]: gc.garbage
    Out[11]: []

gc.garbage为空表示即使出现循环引用, 对象也会被回收掉了

大概的原因是当出现循环引用的时候, 比如上面的a, b, python并不知道应该先调用谁的__del__, 如果调用顺序错了怎么办, 所以决定上面都不做(do nothing)~~


python3.4之后这个问题就解决了, 下面的代码是python3.6

.. code-block:: python

    In [2]: import gc
    
    In [3]: class T:
       ...:     def __del__(self):
       ...:         print('in T __del__%s' % self.name)
       ...:         
    
    In [4]: a=T()
    
    In [5]: b=T()
    
    In [6]: a.name='a'
    
    In [7]: b.name='b'
    
    In [8]: a.other = b
    
    In [9]: b.other = a
    
    In [10]: a=b=None
    
    In [11]: gc.collect()
    in T __del__a
    in T __del__b
    Out[11]: 60

https://www.python.org/dev/peps/pep-0442/ 中有具体细节


内存分配
===========

http://www.wklken.me/posts/2014/08/06/python-source-int.html

http://deeplearning.net/software/theano/tutorial/python-memory-management.html

http://www.wklken.me/posts/2015/09/29/python-source-gc.html

关于整数回收, python是不会把整数的内存放回os的, 是放回自己维护的几个整数池的, 只有python解释器终止了才会释放.


python weakref
===============

https://docs.python.org/3/library/weakref.html

https://segmentfault.com/a/1190000005729873

In the following, the term referent means the object which is referred to by a weak reference.

下面提到的 **引用对象** 是指的是一个 **被弱引用对象引用** 的对象

A weak reference to an object is not enough to keep the object alive: when the only remaining references to a referent are weak references, garbage collection is free to destroy the referent and reuse its memory for something else.

However, until the object is actually destroyed the weak reference may return the object even if there are no strong references to it.

一旦一个对象只有弱引用指向它, 那么它随时可以被回收

weakref是创建一个到目标对象的弱引用, 创建的弱引用不会增加对象的引用计数:

.. code-block:: python

    In [5]: import weakref
    
    In [6]: class M:
       ...:     def __init__(self, name):
       ...:         self.name = name
       ...:         
    
    In [7]: x=M('x')
    
    In [8]: import sys
    
    In [9]: sys.getrefcount(x)
    Out[9]: 2
    
    In [10]: r=weakref.ref(x)
    
    In [11]: r
    Out[11]: <weakref at 0x7fc279f1ad08; to 'instance' at 0x7fc279e9d7e8>
    
    In [12]: sys.getrefcount(x)
    Out[12]: 2


__slots__
=============

https://docs.python.org/3/reference/datamodel.html#slots

https://stackoverflow.com/questions/472000/usage-of-slots


简单来说就是__slots__限定了python对象不能随意添加属性, 属性只能使用定义在__slots__中的元素, 这和__dict__区分开了, __dict__是你添加一个属性就会在__dict__保存起来

可以任意添加属性, 而__slots__不能. 然后可以在__slots__中定义__dict__, 不过这样看起来怪怪的了.

好处的话就是 **对象属性的快速访问和节省内存**, 节省内存是因为属性被限定了, 所以可以省.


.. code-block:: python

    In [1]: class SLOTS:
       ...:     __slots__ = ['data']
       ...:     pass
       ...: 
    
    In [2]: s=SLOTS()
    
    In [3]: s.data
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-3-787769582ab0> in <module>()
    ----> 1 s.data
    
    AttributeError: data
    
    In [4]: s.data=1
    
    In [5]: s.data
    Out[5]: 1
    
    In [6]: s.name='s'
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-6-34c92dadf376> in <module>()
    ----> 1 s.name='s'
    
    AttributeError: 'SLOTS' object has no attribute 'name'



class
=========

property
-----------

https://stackoverflow.com/questions/17330160/how-does-the-property-decorator-work

https://docs.python.org/3/howto/descriptor.html

property返回一个property对象, property对象是属于data descriptor的数据模型, data descriptor是这样一个定义了__get__, __set__, __delete__方法的对象

.. code-block:: python

    In [1]: import inspect
    
    In [2]: def my_getter(self): return 'in my_getter'
    
    In [3]: prop = property(my_getter)
    
    In [4]: type(prop)
    Out[4]: property
    
    In [5]: inspect.isdatadescriptor(prop)
    Out[5]: True


data descriptor的特点呢, 就是如果一个 **某个类的属性是data descriptor**, 那么当访问 **这个类** 该属性的时候, 会调用data descriptor的__get__方法, setter, deleter同理.


.. code-block:: python

    In [13]: class C:
        ...:     x = prop
        ...:     
    
    In [14]: C.x
    Out[14]: <property at 0x7fec39a68318>
    
    In [15]: c=C()
    
    In [16]: c.x
    Out[16]: 'in my_getter'


可以看到, 定义C类中的一个属性x为property(data descriptor), 然后如果用C.x访问，就是一个property对象, 如果是实例c访问, c.x, 那么会直接调用prop这个property(data descriptor)

的__get__方法, 会调用property对象定义的getter方法, 也就是my_getter函数


要注意的是, **property(data descriptor)只有定义在类中才有用!!**


.. code-block:: python

    In [17]: def my_getter(self): return 'in my_getter'
    
    In [18]: prop = property(my_getter)
    
    In [19]: class C:
        ...:     def __init__(self):
        ...:         self.x = prop
        ...:         
    
    In [20]: C.x
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-20-64cce3573e3e> in <module>()
    ----> 1 C.x
    
    AttributeError: type object 'C' has no attribute 'x'
    
    In [21]: c=C()
    
    In [22]: c.x
    Out[22]: <property at 0x7fec3872b408>


上面的例子中, x这个property是在__init__中赋值给实例属性的, 而不是类属性, 所以直接c.x的时候, 得到的是property对象, 而不是期望的in my_getter字符串.

**property只有在类中才能生效, 推测这是因为property的各个魔术方法(__get__, __set__, __delete__)都接收一个instance和owner, owner就是类, 也就是说property是跟类绑定的**

.. code-block:: python

    In [31]: prop.__get__?
    Signature:      prop.__get__(instance, owner, /)
    Call signature: prop.__get__(*args, **kwargs)
    Type:           method-wrapper
    String form:    <method-wrapper '__get__' of property object at 0x7fec3872b408>
    Docstring:      Return an attribute of instance, which is of type owner.
    
    In [32]: prop.__get__(c, C)
    Out[32]: 'in my_getter'

__get__方法接收owner, 也就是一个类, 比如我们直接调用prop.__get__(c, C)就直接调用了定义的getter方法


实例中动态设置类property
~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以动态地在实例中设置类的property, 使用self.__class__就可以了

.. code-block:: python

    In [23]: class C:
        ...:     def __init__(self):
        ...:         c_class = self.__class__
        ...:         c_class.x = prop
        ...:         return
        ...:     
    
    In [24]: C.x
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-24-64cce3573e3e> in <module>()
    ----> 1 C.x
    
    AttributeError: type object 'C' has no attribute 'x'
    
    In [25]: c=C()
    
    In [26]: c.x
    Out[26]: 'in my_getter'


所以, 总结起来就是, property只有类属性的时候才会调用property(data descriptor)对象的__get__方法, 如果某个实例属性被定义为property, 那也是无效的, 访问该属性只能得到property对象

可以在实例中动态地使用self.__class__来将property赋值给类


function
============


内联函数和__closure__
-------------------------

内嵌函数的闭包, 内置函数的__closure__包含了函数定义的时候, 上一层的变量


.. code-block:: python

    In [92]: def outer(a, b):
        ...:     x, y = a, b
        ...:     def inner():
        ...:         a, b = x, y
        ...:         c = a+b
        ...:         return c
        ...:     return inner
        ...: 
    
    In [93]: x=outer(1,2)
    
    In [94]: x
    Out[94]: <function __main__.outer.<locals>.inner>
    
    In [95]: x.__closure__
    Out[95]: 
    (<cell at 0x7fec38fb0228: int object at 0xa61ce0>,
     <cell at 0x7fec38fb0648: int object at 0xa61d00>)
    
    In [96]: x.__closure__[0]
    Out[96]: <cell at 0x7fec38fb0228: int object at 0xa61ce0>
    
    In [97]: a=x.__closure__[0]
    
    In [98]: a.cell_contents
    Out[98]: 1

inner.__closure__保存了在outer定义的局部变量, 如果outer的遍历是在inner定义之后定义的呢? 一样会保存的


.. code-block:: python

    In [104]: def outer(a, b):
         ...:     x, y = a, b
         ...:     def inner():
         ...:         a, b = x, y
         ...:         c = a+b
         ...:         return '%s, %s' % (c, q)
         ...:     q = 'q data'
         ...:     return inner
         ...: 
         ...: 
    
    In [105]: x=outer(1,2)
    
    In [106]: for i in x.__closure__:
         ...:     print(i.cell_contents)
         ...:     
    q data
    1
    2

**如果把outer的遍历直接放到inner里面而不是在inner之前重定义呢?**

.. code-block:: python

    In [146]: def outer(a):
         ...:     def inner():
         ...:         return a, g
         ...:     return inner
         ...: 
    
    In [147]: x=outer(1)
    
    In [148]: x.__closure__
    Out[148]: (<cell at 0x7fec38fb0eb8: int object at 0xa61ce0>,)
    
    In [149]: a=x.__closure__[0]
    
    In [150]: a.cell_contents
    Out[150]: 1
    
    In [151]: x()
    Out[151]: (1, 'g')
 

这样也是可以的, 全部变量自然也可以

默认值
---------

函数的默认值呢, 是在定义的时候就保存到函数对象的__default__里面去了


.. code-block:: python

    In [107]: def outer(a, b):
         ...:     x, y = a, b
         ...:     def inner(inner_default='pp'):
         ...:         a, b = x, y
         ...:         c = a+b
         ...:         return '%s, %s, %s' % (c, q, inner_default)
         ...:     q = 'q data'
         ...:     return inner
         ...: 
         ...: 
         ...: 
    
    In [108]: x=outer(1,2)
    
    In [109]: x.__defaults__
    Out[109]: ('pp',)

如果在函数的内部重新赋值变量名字, 默认值会变么? 当然不会了, 变量赋值一个指向引用的方式, 指向变了, 不会修改之前的对象的(也可以看成变量名是一个盒子, 赋值就是把对象放到盒子里面)


.. code-block:: python

    In [118]: def func(a='a', b='b'):
         ...:     a = 1
         ...:     b = 2
         ...:     return '%s,%s' % (a,b)
         ...: 
    
    In [119]: func.__defaults__
    Out[119]: ('a', 'b')
    
    In [120]: def func(a='3', b='4'):
         ...:     a = 1
         ...:     b = 2
         ...:     return '%s,%s' % (a,b)
         ...: 
    
    In [121]: func.__defaults__
    Out[121]: ('3', '4')
    
    In [122]: func()
    Out[122]: '1,2'

本来a, b指向的是3, 4, 然后func运行的时候a, b指向了1, 2, 当然和默认对象(3, 4)没关系了

**然后如果默认值是一个可变对象的话, 修改是会修改可变对象的, 因为可变对象是in-place修改, 不是重新赋值(assignment)!**


.. code-block:: python

    In [123]: def func(a='a', b=[]):
         ...:     a = 1
         ...:     b.append(1)
         ...:     return '%s,%s' % (a,b)
         ...: 
    
    In [124]: func.__defaults__
    Out[124]: ('a', [])
    
    In [125]: func()
    Out[125]: '1,[1]'
    
    In [126]: func.__defaults__
    Out[126]: ('a', [1])
    
    In [127]: func()
    Out[127]: '1,[1, 1]'
    
    In [128]: func.__defaults__
    Out[128]: ('a', [1, 1])



