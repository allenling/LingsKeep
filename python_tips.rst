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


python dict/list/tuple/set
============================


dict
---------

https://docs.python.org/3/faq/design.html#how-are-dictionaries-implemented

使用开放地址法的变长的hash map

Python’s dictionaries are implemented as resizable hash tables. Compared to B-trees, this gives better performance for lookup (the most common operation by far) under most circumstances, and the implementation is simpler.

插入一个key,value之后的hash map的长度大于总长度的2/3, 则hash map会扩容, 并且把老的hash map的元素使用新的掩码(新的hash map的长度)复制到新的hash map上

hash key的算法参考: http://effbot.org/zone/python-hash.htm

https://stackoverflow.com/questions/327311/how-are-pythons-built-in-dictionaries-implemented

3.6已经重新实现了一个结构更紧凑的dict, 参考了pypy的实现: https://docs.python.org/3/whatsnew/3.6.html#new-dict-implementation, https://morepypy.blogspot.com/2015/01/faster-more-memory-efficient-and-more.html

比起3.5, 节省了20%-25%的内存, 并且现在keys返回是有序的，和key插入的顺序一样, 而3.6之前是ascii顺序, 但是ipython中你回车出来看到的依然是ascii顺序的:

.. code-block:: python

    In [44]: x={'b': 1, 'a': 2}
    
    In [45]: x
    Out[45]: {'a': 2, 'b': 1}
    
    In [46]: list(x.keys())
    Out[46]: ['b', 'a']

dict的key是由对象的hash值决定的, 然后所有的不可变类型都是可以hash的, 然后用户定义的类型的默认hash值是其的id, 比如类T, t=T(), t的hash值就是id(t), 你可以定义一个__hash__方法来决定对象的hash值

https://docs.python.org/3/glossary.html#term-hashable


如何压缩
~~~~~~~~~~~~~~~~~~~~

源码在: https://github.com/python/cpython/blob/v3.6.3/Objects/dictobject.c

首先源码注释中说明了新的dict的结构

.. code-block:: python

    '''
    +---------------+
    | dk_refcnt     |
    | dk_size       |
    | dk_lookup     |
    | dk_usable     |
    | dk_nentries   |
    +---------------+
    | dk_indices    |
    |               |
    +---------------+
    | dk_entries    |
    |               |
    +---------------+
    '''

并且dk_indices是哈希表, 然后dk_entries是key对象的数组, 从dk_indices里面存储的是dk_entries的下标, 然后在3.6的dict实现的建议里面, 有:

For example, the dictionary:

    d = {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}

is currently stored as:

.. code-block:: python

    entries = [['--', '--', '--'],
               [-8522787127447073495, 'barry', 'green'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               [-9092791511155847987, 'timmy', 'red'],
               ['--', '--', '--'],
               [-6480567542315338377, 'guido', 'blue']]

Instead, the data should be organized as follows:

.. code-block:: python

    indices =  [None, 1, None, None, None, 0, None, 2]
    entries =  [[-9092791511155847987, 'timmy', 'red'],
                [-8522787127447073495, 'barry', 'green'],
                [-6480567542315338377, 'guido', 'blue']]

之前的dict和3.6的dict各种接口操作是一样的, 比如都是计算了hash之后, 拿到hash再经过mod运算, 得到hash表的下标, 比如某个key的hash=-8522787127447073495, 这个hash % 8 = 1, 就去entries这个hash表的下标1的数组

去比对hash值, 然后比对相等, 则返回, 如果不相等, 则二次探测. 区别的是3.6的dict中, hash表被单独提出来为indices, 然后原来的hash, key, value这个组合依然存储在entries这个数组内, 然后indices存储的是

entries的下标, 所以3.6的dict是先在计算hash表的下标, 比如hash=-8522787127447073495, 然后hash mod 8 = 1, 然后去查询indices数组下标为1元素, 里面是1, 表示应该去查询entries数组下标为1的元素, 然后去

查找entries数组下标为1的元素, 是一个hash, key, value对, 然后比对hash值, 相等则返回, 不相等, 则继续二次探测. 二次探测为: (next_j = ((5*j) + 1) mod (perturb >> PERTURB_SHIFT), 其中perturb=hash, PERTURB_SHIFT=5.


可以看到, 原来的dict是一个entries就是一个hash表, 然后下标对应存储的是hash值和key, value, 然后存储的空间就很浪费, 64位机器下是24 bit一个hash表的row, 所以之前存储

的话就要花费24 * 8 = 192 bit. 而3.6的dict则是hash数组是int数组, 元素为1 bit来, 表示entries数组的下标, 而 **entries表是一个插入的时候append only的数组**, 是紧凑的数组, 而花费的空间

为: 8(hash数组) + 24 * 3 = 80, 所以空间大幅度减少了.


排序的区别
~~~~~~~~~~~~~

之前的dict是"无序"的, 其实应该说是按hash值排序的, 在之前的dict中, keys的代码为:

https://hg.python.org/cpython/file/52f68c95e025/Objects/dictobject.c#l1180

.. code-block:: c

    static PyObject *
    dict_keys(register PyDictObject *mp) {
        ep = mp->ma_table;
        mask = mp->ma_mask;
        for (i = 0, j = 0; i <= mask; i++) {
            if (ep[i].me_value != NULL) {
                PyObject *key = ep[i].me_key;
                Py_INCREF(key);
                PyList_SET_ITEM(v, j, key);
                j++;
            }
        }
    }


可以看到, 遍历的时候的终止条件是i<=mask, 而mask则是hash表的长度-1, 也就是会遍历hash表, 所以得到的key就是hash排序的key


而3.6的keys函数为:

https://github.com/python/cpython/blob/v3.6.3/Objects/dictobject.c#L2179

.. code-block:: c

    static PyObject *
    dict_keys(PyDictObject *mp)
    {
        ep = DK_ENTRIES(mp->ma_keys);
        size = mp->ma_keys->dk_nentries;
        for (i = 0, j = 0; i < size; i++) {
            if (*value_ptr != NULL) {
                PyObject *key = ep[i].me_key;
                Py_INCREF(key);
                PyList_SET_ITEM(v, j, key);
                j++;
            }
            value_ptr = (PyObject **)(((char *)value_ptr) + offset);
        }
    }

可以看到, size是dk_nentries的大小, 也就是dk_entries的大小, 然后遍历的时候会从ep直接拿key对象的指针, 而ep就是dk_entries, 所以也就是直接遍历dk_entries

这个数组, 而这个数组是insert的时候append only的, 也就是保持了插入的顺序

hash table rbt(map)
~~~~~~~~~~~~~~~~~~~~~

http://blog.csdn.net/ljlstart/article/details/51335687

https://www.zhihu.com/question/24506208

大概就是:

1. hash table内存比较大, map的话内存比较小

2. hash table是无序的, map的话是有序的

3. map比较稳定, 最差也就是logN, hash table好的时候可以说常数级, 但是这个常数级可能比logN大, 然后最坏的时候搜索要遍历整个hash table, 也就是O(N)
   
  也就是hash table搜索效率依赖于冲突, hash table冲突很大的话, 搜索就慢了, 可以打到O(N)



set
------

https://fanchao01.github.io/blog/2016/10/24/python-setobject/


一个hash table实现的, 然后遍历的时候就是遍历hash table, 所以看起来才是"无序"的


list
-------

https://www.laurentluce.com/posts/python-list-implementation/

列表也就是一个数组, 然后当添加或者删除一个元素的时候, 列表的长度会变化的, 下面是代码摘抄:

.. code-block:: c

    // https://github.com/python/cpython/blob/v3.6.3/Objects/listobject.c

    static int list_resize(PyListObject *self, Py_ssize_t newsize)
    {
        PyObject **items;
        size_t new_allocated;
        Py_ssize_t allocated = self->allocated;
    
        /* Bypass realloc() when a previous overallocation is large enough
           to accommodate the newsize.  If the newsize falls lower than half
           the allocated size, then proceed with the realloc() to shrink the list.
        */
        // allocated >> 1这个是allocated / 2, 这样计算二分之一, 可以可以
        // 这里的判断条件中前一个是如果是append, 并且列表本身已经分配的内存足够, 则不需要额外分配内存
        // 第二个判断条件是新的大小, 有可能是长度变小了, 如果还是大于已分配内存的一半, 也不需要缩减内存
        // 所以, 换句话说, 需要扩展内存大小的情况是, newsize大于已分配的内存, 需要缩减内存的情况是
        // newsize的大小小于已分配的一半
        if (allocated >= newsize && newsize >= (allocated >> 1)) {
            assert(self->ob_item != NULL || newsize == 0);
            Py_SIZE(self) = newsize;
            return 0;
        }
    
        /* This over-allocates proportional to the list size, making room
         * for additional growth.  The over-allocation is mild, but is
         * enough to give linear-time amortized behavior over a long
         * sequence of appends() in the presence of a poorly-performing
         * system realloc().
         * The growth pattern is:  0, 4, 8, 16, 25, 35, 46, 58, 72, 88, ...
         */
        // newsize >> 3是newsize往右移３位, 也就是newsize / 8, 毕竟往右移一位等于除以2
        // new_allocated是多分配的大小, new_allocated加上newsize才是上面注释写的步长
        // 比如newsize = 1, 然后 1 >> 3 = 0, 1 < 9, 所以是new_allocated = 0 + 3 =3, newsize = 1
        // 所以是allocated = new_allocated + newsize = 3 +1 = 4
        // 如果是pop等操作的话, allocated会减少, 比如allocated = 8, newsize = 3
        // 则new_allocated = 0 + 3 = 3, 所以最后allocated = new_allocated + newsize = 3 + 3 = 6
        new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6);
    
        /* check for integer overflow */
        if (new_allocated > SIZE_MAX - newsize) {
            PyErr_NoMemory();
            return -1;
        } else {
            new_allocated += newsize;
        }
    
        if (newsize == 0)
            new_allocated = 0;
        items = self->ob_item;
        // 这里的PyMem_RESIZE才是真正的去改变内存大小
        if (new_allocated <= (SIZE_MAX / sizeof(PyObject *)))
            PyMem_RESIZE(items, PyObject *, new_allocated);
        else
            items = NULL;
        if (items == NULL) {
            PyErr_NoMemory();
            return -1;
        }
        self->ob_item = items;
        // 这里self是列表对象, PySIZE(self)是self的长度, 然后这里就赋值为newsize
        Py_SIZE(self) = newsize;
        // 这里赋值列表对象的已分配内存为new_allocated
        self->allocated = new_allocated;
        return 0;
    }

跟位置无关的操作, 比如append, pop的复杂度都是O(1), 其他跟位置有关的都是O(n), 比如insert, pop(index), remove(value)



python interpreter(vm)
========================

https://www.ics.uci.edu/~brgallar/week9_3.html

vs java vm: https://stackoverflow.com/questions/441824/java-virtual-machine-vs-python-interpreter-parlance


python gc
============

http://www.wklken.me/posts/2015/09/29/python-source-gc.html

https://docs.python.org/3/faq/design.html#how-does-python-manage-memory

The standard implementation of Python, CPython, uses reference counting to detect inaccessible objects, and another mechanism to collect reference cycles, periodically

executing a cycle detection algorithm which looks for inaccessible cycles and deletes the objects involved

CPython是引用计数为主, 然后定期去检测循环引用

https://pymotw.com/3/gc/

http://patshaughnessy.net/2013/10/24/visualizing-garbage-collection-in-ruby-and-python

这个文章对比了ruby和python的gc, 里面提到了python还使用了分代回收的机制

https://docs.python.org/3/library/gc.html

上面的gc模块文档中, gc.collect(generation=2)中的generation表示几代, 也就是说python确实也使用了分代回收的gc机制

总结起来就是python中gc是引用计数为主, 然后辅以分代回收, 然后在分代各个代上定时循环检测引用, 包括循环引用. 


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



asynchronous api
=====================

关于协程: https://www.python.org/dev/peps/pep-0492/, async def的coroutine的code_flag是CO_COROUTINE, 而基于生成器的coroutine的话code_flag是CO_ITERABLE_COROUTINE

关于curio的实现: http://dabeaz.com/coroutines/Coroutines.pdf, 基本上是curio的核心思路已经，还有用yield来模仿os多任务切换的例子~~~

https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5


yield
-------

暂停并且返回后面的对象, 使用了yield的函数就是一个生成器.

关于生成器的实现: http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html

yield from
-----------

一个代理语法

yield from a等同于
            
.. code-block:: python

    for item in iterable:
        yield item

执行过程是这样的:

如果是res = yield from iterable, 那么res就是在iterable终止(引发StopIteration)时候的值, yield from 会把iterable的值给yield 到上一层, 所以就是这样:

iterable有yield, 则yield from作为代理, 把值yield 到上一层(这时候功能和yield一样), 然后iterable终止, yield from捕获StopIteration, 然后把StopIteration.value赋值给res.

async/await: 
---------------

之前用yield, yield from的生成器实现的协程和生成器混在一起, 所以才出了async/await, 剥离协程和生成器, 本质和yield, yield from没什么区别, 关于coroutine/awaitable, 文档一句话就很好理解了: https://docs.python.org/3/reference/datamodel.html#coroutines

Coroutine objects are awaitable objects. A coroutine’s execution can be controlled by **calling __await__() and iterating over the result**. When the coroutine has finished executing and returns, the iterator raises StopIteration, and the exception’s value attribute holds the return value. If the coroutine raises an exception, it is propagated by the iterator. Coroutines should not directly raise unhandled StopIteration exceptions.

Coroutines also have the methods listed below, which are analogous to those of generators (see Generator-iterator methods). However, unlike generators, coroutines do not directly support iteration.

不能在一个coroutine上await两次，否则会引发RuntimeError

await如何执行的
-------------------

calling __await__() and iterating over the result这句话解释了协程的工作方式: await obj, 会对obj进行一个迭代!!

我们知道yield是暂停并且返回程序, 然后yield from是一个代理语法，本质上是对yield from后面的对象进行迭代, 而await则是对后面的awaitable对象进行迭代, 直到引发StopIteration, await基本可以看成是
协程中的yield from(因为yield from不能出现在async def里面), **await和yield from执行过程一样**

例子:

.. code-block:: python

    # 一个可迭代对象(实现了__iter__), 同时也是自己的迭代器对象(实现了__next__)
    class Counter:
    
        def __init__(self):
            self.start = 0
            return
    
        def __iter__(self):
            return self
    
        def __next__(self):
            if self.start < 10:
                tmp = self.start
                self.start += 1
                return tmp
            raise StopIteration

然后yield from的例子

.. code-block:: python

    def yield_from_test():
        counter = Counter()
        data = yield from counter
        return data

以及await的例子, **await后面要接一个awaitable对象, 也就是实现了__await__方法的对象, __await__返回一个可迭代对象**


.. code-block:: python

    async def await_test():
        counter_await = CounterAwait
        data = await counter_await
        return data

yield from和await的例子实现的是同样一个功能


用dis来看opcode的区别:

.. code-block:: python

    dis.dis(yield_from_test)
    dis.dis(await_test)

输出分别是, yield_from_test:


.. code-block:: 

    34           0 LOAD_GLOBAL              0 (Counter)
                 2 CALL_FUNCTION            0
                 4 STORE_FAST               0 (counter)
    
    35           6 LOAD_FAST                0 (counter)
                 8 GET_YIELD_FROM_ITER
                10 LOAD_CONST               0 (None)
                12 YIELD_FROM
                14 STORE_FAST               1 (data)
    
    36          16 LOAD_FAST                1 (data)
                18 RETURN_VALUE

以及await_test:

.. code-block::

    28           0 LOAD_GLOBAL              0 (CounterAwait)
                 2 STORE_FAST               0 (counter_await)
    
    29           4 LOAD_FAST                0 (counter_await)
                 6 GET_AWAITABLE
                 8 LOAD_CONST               0 (None)
                10 YIELD_FROM
                12 STORE_FAST               1 (data)
    
    30          14 LOAD_FAST                1 (data)
                16 RETURN_VALUE


**共同点**: 就是都会有YIELD_FROM

**区别就是**:
  
1.yield from的对象, yield from语句基本上就是对后面的可迭代对象进行迭代, GET_YIELD_FROM_ITER就是拿到可迭代对象, 也就是调用__iter__方法

2. 而await呢, 则是有一个GET_AWAITABLE的指令，然后GET_AWAITABLE就是调用__await__方法,获取其返回的对象, 所以await是对__await__返回的对象进行yield from

**所以await obj就是调用obj.__await__方法，得到返回的可迭代对象iter_obj, 然后对得到的可迭代对象进行yield from, yield from是yield的代理, 语句yield from iter_obj, 本质上会对yield from后面的对象iter_obj进行一个迭代
(这里迭代有点小小的困惑, 一直调用iter_obj.send比较合适), 直到产生了StopIteration, 然后捕获StopIteration.value, 返回给调用者.**

之所以上面说迭代有小小的困惑，是因为你可以await coroutine, 但是coroutine是不可以迭代的, 但是同时, coroutine终止的时候却是StopIteration, 就很迷~~~, 所以说还是用send方法表示合适，
因为send对coroutine和可迭代对象都适用.

**所以，也就是说不管是yield from还是await也只是一个语法而已, 要实现coroutine, 关键是如何实现停止，重新执行, pytho中是遇到yield, 然后停止, send重新执行. 所以核心的还是yield**

coroutine也是awaitable对象，也有__await__, coroutine.__await__返回的是一个叫coroutine_wrapper的对象, 这个对象实现了send, close, throw这些方法, 看名字就知道
这个coroutine_wrapper只是对coroutine加了一个包装而已

所以说async/await只是个api, 定义了使用范围等等规范, async定义coroutine, await会暂停，同时await后面的awaitable对象, 至于awaitable对象是什么, 你怎么处理awaitable对象(比如丢弃), 就可以具体发挥了, 比如curio中是一些
自己定义的协程(curio.sleep), 而asyncio是Future对象


StopIteration history
=============================


3.3之前生成器是不能有return, 3.3之后是可以有return的, return var的话会raise StopIteration, 其中StopIteration.value存储了return的值, 这样可以显式地
捕获最后的返回值，然后用yield from也可以方便的拿到最后返回的值了:https://www.python.org/dev/peps/pep-0380/#use-of-stopiteration-to-return-values

.. code-block:: python

    In [61]: def foo():
        ...:     yield 1
        ...:     return 2
        ...: 
    
    In [62]: def bar():
        ...:     v = yield from foo()
        ...:     print(v)
        ...:     
    
    In [63]: x=bar()
    
    In [64]: next(x)
    Out[64]: 1
    
    In [65]: next(x)
    2
    ---------------------------------------------------------------------------
    StopIteration                             Traceback (most recent call last)
    <ipython-input-65-92de4e9f6b1e> in <module>()
    ----> 1 next(x)
    
    StopIteration: 

然后for循环的话是碰到StopIteration的话就停止, 并且忽略StopIteration的

当然，你也可以手动raise StopIteration, 跟return一样的


.. code-block:: python

    >>> def foo():
    ...     yield 1
    ...     raise StopIteration(2)
    ... 
    >>> def f():
    ...     v = yield from foo()
    ...     print(v)
    ...     return
    ... 
    >>> x=f()
    >>> x
    <generator object f at 0x7f6607bcc0f8>
    >>> next(x)
    1
    >>> next(x)
    2
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    StopIteration
    >>> 


**但是3.6开始, 在生成器中手动raise StopIteration的话会报警告, 之后应该是不允许在生成器中手动raise StopIteration了, https://www.python.org/dev/peps/pep-0479/**

同理, coroutine中return也是会raise StopIteration的, 这个和生成器一样.



asynchronous iterator
==========================

关于asynchronous iterator: https://www.python.org/dev/peps/pep-0492/#id62

An asynchronous iterable is able to call asynchronous code in its iter implementation, and asynchronous iterator can call asynchronous code in its next method. To support asynchronous iteration:

1. An object must implement an __aiter__ method (or, if defined with CPython C API, tp_as_async.am_aiter slot) returning an asynchronous iterator object.

2. An asynchronous iterator object must implement an __anext__ method (or, if defined with CPython C API, tp_as_async.am_anext slot) returning an **awaitable**.
   这里说了, __anext__必须返回一个awaitable对象

3. To stop iteration __anext__ must raise a StopAsyncIteration exception.

也就是说async for一个async iterator的时候，只有遇到StopAsyncIteration才会停止迭代，跟同步的iterator差不多

ok, 用例子回顾一下iterator:

.. code-block:: python

    In [18]: class it:
        ...:     def __init__(self, n):
        ...:         self.n = n
        ...:     def __iter__(self):
        ...:         return self
        ...:     def __next__(self):
        ...:         if self.n:
        ...:             tmp = self.n
        ...:             self.n -= 1
        ...:             return tmp
        ...:         else:
        ...:             raise StopIteration('im done')
        ...:          
    
    In [19]: i=it(2)
    
    In [20]: for j in i:
        ...:     print(j)
        ...:     
    2
    1


对于asynchronous iterator, 也一样, 只是StopIteration换成了StopAsyncIteration,

.. code-block:: python

    class ait:
        def __init__(self, to):
            self.to = to
            self.start = 0
    
        def __aiter__(self):
            return self
    
        async def __anext__(self):
            if self.start < self.to:
                tmp = self.start
                self.start += 1
                return tmp
            else:
                raise StopAsyncIteration
    
    a = ait(2)
    
    an = a.__anext__()

    print('an is: %s' % an)
    
    try:
        v = an.send(None)
    except StopIteration as e:
        print(e.value)

输出是

an is: <coroutine object ait.__anext__ at 0x7f216ebe80f8>

0

dis一下:

.. code-block:: python

    import dis
    
    
    class ait:
        def __init__(self, to):
            self.to = to
            self.start = 0
    
        def __aiter__(self):
            return self
    
        async def __anext__(self):
            if self.start < self.to:
                tmp = self.start
                self.start += 1
                return tmp
            else:
                raise StopAsyncIteration
    
    
    async def test():
        async for i in ait(2):
            print(i)
    
    
    dis.dis(test)

输出是

.. code-block:: 

 22           0 SETUP_LOOP              62 (to 64)
              2 LOAD_GLOBAL              0 (ait)
              4 LOAD_CONST               1 (2)
              6 CALL_FUNCTION            1
              8 GET_AITER
             10 LOAD_CONST               0 (None)
             12 YIELD_FROM
        >>   14 SETUP_EXCEPT            12 (to 28)
             16 GET_ANEXT
             18 LOAD_CONST               0 (None)
             20 YIELD_FROM
             22 STORE_FAST               0 (i)
             24 POP_BLOCK
             26 JUMP_FORWARD            22 (to 50)
        >>   28 DUP_TOP
             30 LOAD_GLOBAL              1 (StopAsyncIteration)
             32 COMPARE_OP              10 (exception match)
             34 POP_JUMP_IF_FALSE       48
             36 POP_TOP
             38 POP_TOP
             40 POP_TOP
             42 POP_EXCEPT
             44 POP_BLOCK
             46 JUMP_ABSOLUTE           64
        >>   48 END_FINALLY
 
 23     >>   50 LOAD_GLOBAL              2 (print)
             52 LOAD_FAST                0 (i)
             54 CALL_FUNCTION            1
             56 POP_TOP
             58 JUMP_ABSOLUTE           14
             60 POP_BLOCK
             62 JUMP_ABSOLUTE           64
        >>   64 LOAD_CONST               0 (None)
             66 RETURN_VALUE

注意的是, GET_AITER就是调用__aiter__方法, 然后GET_ANEXT就是调用__anext__方法, 然后接着有一个YIELD_FROM, 也就是对__anext__返回的对象进行yield from, 然后跳到print那里打印, 然后
LOAD_GLOBAL StopAsyncIteration, 然后COMPARE_OP比对异常是否符合, 如果符合就是END_FINALLY


所以，从流程上看, asynchronous iterator基本上是调用__anext__, 如果没有遇到StopAsyncIteration异常，则返回一个awaitable对象, 例子中就是an, an是一个coroutine(因为是async def), 也是一个awaitable对象.
然后由迭代的对象(比如async for, 各种event pool)对返回回来的awaitable对象一直send, 直到发生StopIteration, 然后这个StopIteration.value就是这一轮迭代的值

比如上面的例子, 调用__anext__返回一个await对象(an), 然后对an进行send, 在an中
会return tmp, 此时tmp=0, 并且return会引发一个StopIteration异常, StopIteration.value就是返回值, 所以v就是0, 调用an.send也是async for的流程, 这里只是手动调用模拟了而已, 所以
完整的流程应该是:

.. code-block:: python

    while True:
        an = a.__anext__()
        try:
            an.send(None)
        except StopIteration as e:
            print(e.value)
        except StopAsyncIteration:
            break

如果an.send(None)不是引发StopIteration呢，假设ares = an.send(None), 如果ares有值, 也就是an被暂停了(调用了await)，根据python中的行为, 则for会把ares返回给上一级, 然后一级一级地
传递, 类似于yield from, 最后传给curio, 或者asyncio, 这样流程是不是很想yield from, 在async def中, 也就是await了, 所以也就是:

.. code-block:: python

    async def show_async_for(asyn_gen):
        async for i in asyn_gen:
            print(i)
    
    
    # 下面这个while循环就是模拟async for的执行
    # 也就是文档中对async for语法的说明
    async def simulation_for(asyn_gen):
        while True:
            anext = asyn_gen.__anext__()
            try:
                value = await anext
                print(value)
            except StopAsyncIteration:
                break

show_async_for().send(None)和simulation_for().send(None)结果都是一样的，最后都会引发StopIteration异常, 因为两者执行完for循环之后都结束了, 所以这个StopIteration是外层
coroutine终止引发的, 不是coroutine内的for循环造成的.

asynchronous generator
--------------------------

https://www.python.org/dev/peps/pep-0525/

3.7才有aiter内建方法

**3.6.3之前, yield 是不能出现在async def中的, 之后是可以了, 并且非空的return不能出现在异步生成器里面**

The protocol requires two special methods to be implemented:

An __aiter__ method returning an asynchronous iterator.
An __anext__ method returning an awaitable object, which **uses StopIteration exception to "yield" values**, and StopAsyncIteration exception to signal the end of the iteration.

比如我们需要实现一个带有delay的计数器, 如果只是使用asynchronous iterator的话，可以这么做:

.. code-block:: python

    class Ticker:
        def __init__(self, delay, to):
            self.to = delay
            self.start = 0
            self.to = to
    
        def __aiter__(self):
            return self
    
        async def __anext__(self):
            if self.start < self.to:
                if self.start > 0:
                    await asyncio.sleep(self.delay)
                tmp = self.start
                self.start += 1
                return tmp
            else:
                raise StopAsyncIteration
    
    
    async def test():
        async for i in ait(2):
            print(i)
    
如果用异步生成器的语法就是

.. code-block:: python

    async  def ticker(delay, to):
        for i in range(to):
            yield i
            await asyncio.sleep(delay)

看看ticker字节码:


.. code-block:: python

  6           0 SETUP_LOOP              38 (to 40)
              2 LOAD_GLOBAL              0 (range)
              4 LOAD_FAST                1 (to)
              6 CALL_FUNCTION            1
              8 GET_ITER
        >>   10 FOR_ITER                26 (to 38)
             12 STORE_FAST               2 (i)
  
  7          14 LOAD_FAST                2 (i)
             16 YIELD_VALUE
             18 POP_TOP
  
  8          20 LOAD_GLOBAL              1 (curio)
             22 LOAD_ATTR                2 (sleep)
             24 LOAD_FAST                0 (delay)
             26 CALL_FUNCTION            1
             28 GET_AWAITABLE
             30 LOAD_CONST               0 (None)
             32 YIELD_FROM
             34 POP_TOP
             36 JUMP_ABSOLUTE           10
        >>   38 POP_BLOCK
        >>   40 LOAD_CONST               0 (None)
             42 RETURN_VALUE

正常的GET_ITER, FOR_ITER, 然后YIELD_VALUS, 遇到await，执行GET_AWAITABLE, 然后YIELD_FROM, 但是文档说的uses StopIteration to "yield" values是怎么体现的?

先看看一般的生成器:

.. code-block:: python

    def normal_gen():
        data = yield 1
        return data
    
    
    ng = normal_gen()
    
    res = ng.send(None)
    
    print(res)
    
    ng.send('d')

一般的生成器是yield之后，你可以再send进去重新启动的. 而异步生成器，是要遵循异步迭代协议的, 也就是有__aiter__和__anext__的, 所以__anext__返回的必然也是一个awaitable, 这里
异步生成器返回的也是awaitable对象, 然后对这个awaitable对象进行迭代(这里可以说迭代, 因为这里主要是迭代协议), 直到发生StopIteration, 捕获然后返回StopIteration.value, StopIteration.value
就是一次迭代的值了, 那么如果是通过引发StopIteration去yield值, 那么如何传入值到异步生成器呢?

.. code-block:: python

    async def ticker():
        data = yield 1
        print('got data %s' % data)
        data = yield 2
        print('second data: %s' % data)
    
    t = ticker()
    
    res = t.asend(None)

    print(res)
    
    try:
        res.send(None)
    except StopIteration as e:
        print(e.value)
    
    res = t.asend('outer data')
    print('-----send res-----')
    try:
        res.send(None)
    except StopIteration as e:
        print(e.value)

这里输出是

.. code-block:: python

   <async_generator_asend object at 0x7f575b2aa7c8>
   1
   -----send res-----
   got data outer data
   2

所以, 可以使用asend方法, 并且，注意的是yield的话直接就是通过引发StopIteration来返回值，这一点遵循异步迭代协议, 然后补充asend方法来传值.

但是, 其实也可以使用res.send来传值, 并且这种传值的方式和asend冲突

.. code-block:: python

    async def ticker():
        data = yield 1
        print('got data %s' % data)
        data = yield 2
        print('second data: %s' % data)
    
    t = ticker()
    
    res = t.asend(None)

    print(res)
    
    try:
        res.send(None)
    except StopIteration as e:
        print(e.value)
    
    res = t.asend('outer data')
    print('-----send res-----')
    try:
        res.send(None)
    except StopIteration as e:
        print(e.value)

输出是:

.. code-block:: python

    <async_generator_asend object at 0x7ff83ceca788>
    1
    -----send res-----
    got data second outer data
    2

outer data被second outer data覆盖了


**通过res来传值是在文档中所提出的实现细节中说明的**:

.. code-block:: python

    async def ticker():
        data = yield 1
        print('got data %s' % data)
        data = yield 2
        print('second data: %s' % data)
    
    t = ticker()
    
    res = t.asend(None)
    
    print(res)
    
    try:
        res.send(None)
    except StopIteration as e:
        print(e.value)
    
    res = t.asend(None)
    
    print('-----send res-----')
    try:
        res.send('outer data')
    except StopIteration as e:
        print(e.value)
    
    res = t.asend(None)
    print('+++++send res++++++')
    try:
        res.send('second outer data')
    except StopIteration as e:
        print(e.value)
    except StopAsyncIteration as e:
        print('async gen done')

输出就是:

.. code-block:: 

    <async_generator_asend object at 0x7f3ae80487c8>
    1
    -----send res-----
    got data outer data
    2
    +++++send res++++++
    second data: second outer data
    async gen done

PyAsyncGenASend and PyAsyncGenAThrow are awaitables (they have __await__ methods returning self) and are coroutine-like objects (implementing __iter__, __next__, send() and throw() methods). Essentially, they control how asynchronous generators are iterated

async_generator_asend对象是类coroutine对象, 控制了异步生成器的迭代过程.

agen.asend(val) and agen.__anext__() return instances of PyAsyncGenASend (which hold references back to the parent agen object.)

async_generator_asend对象保存了指向async_generator对象的指针

When PyAsyncGenASend.send(val) is called for the first time, val is pushed to the parent agen object (using existing facilities of PyGenObject.)

Subsequent iterations over the PyAsyncGenASend objects, push None to agen.

When a _PyAsyncGenWrappedValue object is yielded, it is unboxed, and a StopIteration exception is raised with the unwrapped value as an argument.

调用async_generator_asend.send(val)的时候, val会被传入到async_generator中, 然后顺手遍历了async_generator_asend对象, 遇到yield的时候, async_generator_asend则引发一个StopIteration,
并且StopIteration.value就是yield的值


异步生成器的终止(finalization)
--------------------------------

必须保证就算异步生成器没有全部迭代完, 迭代器的终止也要执行, 这样当gc回收的时候才能正确回收

Asynchronous generators can have try..finally blocks, as well as async with. It is important to provide a guarantee that, even when partially iterated, and then garbage collected, generators can be safely finalized.




