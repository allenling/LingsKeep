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


变量赋值
---------

python的变量赋值其实是一个名字指向一个值, 也就是所谓的引用.

比如a=1, 将a这个名字指向1, 然后a=1, 也是将a这个名字指向1, 在python中, 这个时候是复用了常量1, 所以id(x) == id(a)

然后x=1, a=x的情况是, 名字x指向1, 然后a指向x所指向的值, 也就是1, 所以还是id(x) == id(a), 若x=2, 这个时候x就指向另外一个常量2, 所以x为2, a还是为1, 这个时候id(x)并不等于id(a)

对于字符串和数字之类的不可变对象(包括tuple), 修改会rebind, reassign一个新的值, 比如

x='a'
v1 = id(x)
x += 'b'
v2 = id(x)

这个时候v1和v2并不像等, 也就是其实是x这个名字被指向一个新的字符串了. 同理, x=1, x+=1的情况也一样.

而对于可变对象, 也就是支持modify in place, 也就是可修改的对象, 修改并不会rebind, reassign, 是在原处修改, 除非手动rebind, reassign, 否则指向的值(也就是可变对象)不变.

如

x = [1]
v1 = id(x)
x.append(2)
v2 = id(x)

v1和v2是相等的, 只是原处修改了值, 名字x指向的值没有变.

对于

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

def test(a, b=[]):
    b.append(1)
    print a, b

test(1)
1, [1]
test(1)
1, [1, 1]
test(1)
1, [1, 1, 1]

一开始,
test.func_defaults为([],), test.func_code.co_varnames为('a', 'b')
调用第一次之后, 
test.func_defaults为([1],)
调用第二次之后, 
test.func_defaults为([1, 1],)
以此类推

若默认值不是可变对象, 则存在func.func_defaults的值是不会变的


def test(a, b=1):
    b += 1
    print a, b

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

is或者==
------------

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


python"定时任务
-----------------

一般是100个字节码指令之后去检查信号或者切换线程等.
https://docs.python.org/2/library/sys.html#sys.setcheckinterval

sys.setcheckinterval(interval)
Set the interpreter’s “check interval”. This integer value determines how often the interpreter checks for periodic things such as thread switches and signal handlers. The default is 100, meaning the check is performed every 100 Python virtual instructions. Setting it to a larger value may increase performance for programs using threads. Setting it to a value <= 0 checks every virtual instruction, maximizing responsiveness as well as overhead.


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
    print '--------------------------------------------'
    dis.dis(test1)
    print '--------------------------------------------'
    dis.dis(test_mutable)

输出

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
--------------------------------------------
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
--------------------------------------------
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



https://docs.python.org/2/faq/programming.html#what-are-the-rules-for-local-and-global-variables-in-python

In Python, variables that are only referenced inside a function are implicitly global. If a variable is assigned a value anywhere within the function’s body, it’s assumed to be a local unless explicitly declared as global.

Though a bit surprising at first, a moment’s consideration explains this. On one hand, requiring global for assigned variables provides a bar against unintended side-effects. On the other hand, if global was required for all global references, you’d be using global all the time. You’d have to declare as global every reference to a built-in function or to a component of an imported module. This clutter would defeat the usefulness of the global declaration for identifying side-effects.

关键在于上面句子中的 If a variable is assigned a value anywhere within the function’s body, it’s assumed to be a local unless explicitly declared as global.这句话中的If a variable is assigned, 是assigned的时候
才会当成局部变量.

所以可以这么理解, 在函数里面, 修改字典的时候, 是modify in place, 不是reassign, rebinding, 所以解释器会直接根据LEGB原则加载到全局的字典, 然后修改.

而其他不可变对象就不行, 不可变对象在函数里面若有修改操作, 也就是reassign, rebinding操作, 解释器就把它当成局部变量, 所以第一句打印语句就报没有定义的错误.

有时候得抠字眼一下才能理解.

**一个gotcha情况:**

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


python 3.x
============


sorted(list.sort)
--------------------

1. key函数: 取消了cmp而使用key, 在python2.x之前, cmp会每遍历一个元素就调用一次, 而指定key函数之后, key函数只运行一次，然后根据key返回的值列表排序，很多数据的时候比起cmp, 性能会更好点
   
   https://www.zhihu.com/question/30389643 

   The value of the key parameter should be a function that takes a single argument and returns a key to use for sorting purposes. This technique is fast because the key function is called exactly once for each input record.
   (https://docs.python.org/3/howto/sorting.html)

2. timsort算法: The Timsort algorithm used in Python does multiple sorts efficiently because it can take advantage of any ordering already present in a dataset.

   https://bindog.github.io/blog/2015/03/30/use-formal-method-to-find-the-bug-in-timsort-and-lunar-rover/

3. key中使用operation.itemgetter/attrgetter/methodcaller更快/更好/更清楚, why?

1. python基本类型的比较都是C级别的, 而对象级别的(__eq__等方法)都是python基本的

2. The operator module in python implements many functions of common use in C, making them faster.(http://bioinfoblog.it/2010/01/operator-itemgetter-rocks/), 但是有个operator.py, 你说是C实现的，表示怀疑

3. 有些人说用itemgetter比起lamba, 能更清楚表示在干嘛

4. 好吧，网上都说operator里面的函数比较快，但是为什么快呢，有些人说是C实现的，但是itemgetter的实现明显是python代码, 而且官方文档也没说明，就一句
   The key-function patterns shown above are very common, so Python provides convenience functions to make accessor functions easier and faster. The operator module has itemgetter(), attrgetter(), and a methodcaller() function.
   (https://docs.python.org/3/howto/sorting.html)


django的qs和redis混排
-------------------------


qs.orderby之后搜索出来的就是有序的，然后使用timsort来排序redis中拿到的数据，刚刚好, 前提是qs出来的不是很大~~~


