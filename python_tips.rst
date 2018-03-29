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

`yield的实现 <http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html>`_, yield的作用是返回值并挂起

例如: 生成器产生斐波拉契数列

.. code-block:: python

    import itertool


    def F():
        a,b = 0,1
        yield a
        yield b
        while True:
            a, b = b, a + b
            yield b
        return

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

.. code-block:: python

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

flush socket
----------------

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


__str__和__repr__进行格式化
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

StringIO
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

__slots__属性
===============

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
---------------------------

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

inner.__closure__保存了在outer定义的局部变量, 如果outer的变量是在inner定义之后定义的呢? 一样会保存的


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
 

这样也是可以的, 全局变量自然也可以


logging
=========

logging模块是线程安全的, 因为在basicConfig, getLogger, 以及每一个handler中都有锁

basicConfig, getLogger是全局锁, 而handler是针对handler的锁

basicConfig
---------------

.. code-block:: python

    def basicConfig(**kwargs):
        _acquireLock()
        try:
            # 省略了代码
            pass
        finally:
            _releaseLock()

logging.basicConfig保证了新建和配置某个logger的时候, 是只有一个线程能成功

这里的锁是全局的, 保证了多个线程都配置同一个logger的话, 只有一个能成功

logging.getLogger也是全局锁, 也就是多个线程获取同一个logger的话, 只有一个能成功

logging.getLogger最后调用到Manager.getLogger

.. code-block:: python

    class Manager(object):
        def getLogger(self, name):
            # 省略了代码
            _acquireLock()
            try:
                # 省略了代码
                pass
            finally:
                _releaseLock()
            return rv


handler
---------

比如StreamHandler:

.. code-block:: python

    class StreamHandler(Handler):
        def flush(self):
            """
            Flushes the stream.
            """
            self.acquire()
            try:
                if self.stream and hasattr(self.stream, "flush"):
                    self.stream.flush()
            finally:
                self.release()
    
    class Handler(Filterer):
        def acquire(self):
            """
            Acquire the I/O thread lock.
            """
            # 获取一下锁
            if self.lock:
                self.lock.acquire()

所以每次调用flush的时候, 都会去拿self.lock这个锁

