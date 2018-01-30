变量赋值
=========

**python的变量赋值其实是一个名字指向一个值, 也就是所谓的引用**.

比如a=1, 将a这个名字指向1, 然后a=1, 也是将a这个名字指向1, 在python中, 这个时候是复用了常量1, 所以id(x) == id(a)

然后x=1, a=x的情况是, 名字x指向1, 然后a指向x所指向的值, 也就是1, 所以还是id(x) == id(a), 若x=2, 这个时候x就指向另外一个常量2, 所以x为2, a还是为1, 这个时候id(x)并不等于id(a)

对于字符串和数字之类的不可变对象(包括tuple), 修改会rebind, reassign一个新的值, 比如

.. code-block:: python

    x='a'
    v1 = id(x)
    x += 'b'
    v2 = id(x)

这个时候v1和v2并不像等, 也就是其实是x这个名字被指向一个新的字符串了, x+='b'如下:

.. code-block:: python

   tmp=x
   # 一个新的字符串
   tmp = tmp + 'b'
   x = tmp

同理, x=1, x+=1的情况也一样.

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

函数默认值
===============

这里可以拓展到使用可变对象作为函数默认值的情况, 由于python的函数默认值是在函数定义的时候就赋值了, 而不是每次执行的时候赋值

"Python’s default arguments are evaluated once when the function is defined, not each time the function is called"

--- http://docs.python-guide.org/en/latest/writing/gotchas/

其实可以这么理解, 一旦函数定义了, 函数对象自己就将默认值存起来, 存储到func.__defaults__上了, __defaults__存储的是第n个默认值的值


例如:

.. code-block:: python

    def test(a, b=[]):
        b.append(1)
        print(a, b)
        return

调用和输出:

.. code-block:: 

    In [156]: test(1)
    1 [1]
    
    In [157]: test(1)
    1 [1, 1]
    
    In [158]: test(1)
    1 [1, 1, 1]
    
    In [159]: test(1)
    1 [1, 1, 1, 1]

    In [160]: test.__defaults__
    Out[160]: ([1, 1, 1, 1],)

可以看到func.__defaults__是一个tuple, 其中b=func.__defaults__[0], 然后每次b.append(1), 就是func.__defaults__[0].append(1)

如果默认值是不可变对象:

.. code-block:: python

    def test(a, b=1):
        b += 1
        print(a, b)
        return

调用和输出:

.. code-block:: python

    In [226]: test.__defaults__
    Out[226]: (1,)
    
    In [227]: test(1)
    1 2
    
    In [228]: test(1)
    1 2
    
    In [229]: test(1)
    1 2
    
    In [230]: test.__defaults__
    Out[230]: (1,)

无论调用几次, test.__defaults__都是(1, )

这是因为func.__defaults__是一个元组，不能改变元组的值，但是你可以改变元组里面可变对象的值.

上面的过程就是:

1. 从test.__defaults__中取出默认值, 也就是b = test.__defaults__[0]

2. 然调用b.append(1), 然后其实test.__defaults__[0]指向的是b, 所以b改变的时候是modify in-place, 导致test.__defaults__[0]就改变了

如果你改变了test.__defaults__, 那么test执行会变化吗? 答案是会的:

.. code-block:: python

    In [242]: def test(a, b=1):
         ...:     b += 1
         ...:     print(a, b)
         ...:     return
         ...: 
    
    In [243]: test(1)
    1 2
    
    In [244]: test.__defaults__ = (10, 11)
    
    In [245]: test(1)
    1 12
    
    In [246]: test()
    10 12

看起来, 查询__defaults__的时候有点奇怪~~~

所以, 动态修改默认值是可以的, 比如append. 如果你把b.append(1)换成b+=[1]也是一样的可以的, 但是如果是test.__defaults__[0] += [1]这样就报错: tuple不能改变, 这是为什么呢?

这就引出了下面的问题

tuple的问题
=================

.. code-block:: python

    def test():
        x = ([], 'a')
        x[0].append(1)
        print(x)
        x[0] += [2]
        print(x)
        return

输出的话是, 第一个append执行成功, 第二个+=是失败的, 报错是tuple不能被修改.

第一次x[0].append并没有改变x[0]的值, 改变的意思是reassign, 改变id, 而可变对象的修改是modify in-place的, 并没有reassign, 所以第一次修改成功,

第二次修改是调用了+=这个操作符号, +=对于不可变对象来说, 就是重新赋值的意思, 下面的例子, x+=之后, id并不一样:

.. code-block:: python

    In [80]: x=1
    
    In [81]: id(x)
    Out[81]: 10886368
    
    In [82]: x+=1
    
    In [83]: id(x)
    Out[83]: 10886400

但是+=对于可变变量, 则还是modify in-place, 下面的例子, x的id并没有改变:

.. code-block:: python

    In [85]: x=[1]
    
    In [86]: id(x)
    Out[86]: 140174011130120
    
    In [87]: x+=[2]
    
    In [88]: x
    Out[88]: [1, 2]
    
    In [89]: id(x)
    Out[89]: 140174011130120

看字节码:

.. code-block:: 

  5           0 BUILD_LIST               0
              2 LOAD_CONST               1 ('a')
              4 BUILD_TUPLE              2
              6 STORE_FAST               0 (x)
  
  6           8 LOAD_FAST                0 (x)
             10 LOAD_CONST               2 (0)
             12 BINARY_SUBSCR
             14 LOAD_ATTR                0 (append)
             16 LOAD_CONST               3 (1)
             18 CALL_FUNCTION            1
             20 POP_TOP
  
  7          22 LOAD_GLOBAL              1 (print)
             24 LOAD_FAST                0 (x)
             26 CALL_FUNCTION            1
             28 POP_TOP
  
  8          30 LOAD_FAST                0 (x)
             32 LOAD_CONST               2 (0)
             34 DUP_TOP_TWO
             36 BINARY_SUBSCR
             38 LOAD_CONST               4 (2)
             40 BUILD_LIST               1
             42 INPLACE_ADD
             44 ROT_THREE
             46 STORE_SUBSCR
  
  9          48 LOAD_GLOBAL              1 (print)
             50 LOAD_FAST                0 (x)
             52 CALL_FUNCTION            1
             54 POP_TOP
  
  10          56 LOAD_CONST               0 (None)
             58 RETURN_VALUE

第一个append是直接调用方法的LOAD_FAST拿到x和0, 拿到x[0], 也就是list, 然后调用append

第二个+=是先拿到x和常量0, 然后进行BINARY_SUBSCR来获取x[0]的值, 加载变量2, 然后创建[2]这样一个list, 然后INPLACE_ADD, 也就是修改x[0]指向的list, 最后调用
STORE_SUBSCR来实现x[0] = list(这个list是修改过的), 直白一点就是:

1. BINARY_SUBSCRx: 先拿出tmp=x[0]

2. BUILD_LIST: tmp_list = [2].

3. INPLACE_ADD: tmp = tmp + tmp_list, 这里调用tmp的__iadd__方法, 对应例子中tmp是list, 也就是调用list.__iadd__方法

4. STORE_SUBSCR: x[0] = tmp, 因为x[0]=tmp这个操作就是setitem操作, 也就是修改x的值, 但是x是tuple, 所以报错.

但是由于tmp+=[2]这个操作是直接对x[0]进行modify in-place, 所以是成功的, 报错是把值set回tuple的时候.

所以可知, 只要不涉及到tuple的setitem, 就不会报错

.. code-block:: python

   x = ([], 1)
   # 这个可以
   x[0].append(1)

   # 下面也可以, 没有涉及到setitem
   tmp = x[0]
   tmp += [2]

   # 涉及到setitem, 必然报错
   x[0] += [3]

tuple的C代码在 `这里 <https://github.com/allenling/LingsKeep/blob/master/python_container.rst#tuple>`_

global
========

参考自: https://docs.python.org/2/faq/programming.html#what-are-the-rules-for-local-and-global-variables-in-python

例子:

.. code-block:: python

    d = {'a': 1}
    
    x = 'data'
    
    
    def test():
        print(x, d)
        return

这个时候输出是正常的, 因为, 这是因为python函数中的变量是隐式全局的, 所以x和d都会被当做全局变量:

*In Python, variables that are only referenced inside a function are implicitly global.*

然后, 我们在test中修改一下x, 再输出:

.. code-block:: python

    d = {'a': 1}
    
    x = 'data'
    
    
    def test():
        print(x, d)
        x = 1
        print(x, d)
        return

这个时候报错, x使用之前未定义, 也就是说此时x被当成了局部变量. 可以看看字节码的区别:

第一次例子的dis结果:


.. code-block:: 

    9           0 LOAD_GLOBAL              0 (print)
                2 LOAD_GLOBAL              1 (x)
                4 LOAD_GLOBAL              2 (a)
                6 CALL_FUNCTION            2
                8 POP_TOP
    
    10          10 LOAD_CONST               0 (None)
                12 RETURN_VALUE


第二次修改x再打印的例子的dis:


.. code-block:: 

    9           0 LOAD_GLOBAL              0 (print)
                2 LOAD_FAST                0 (x)
                4 LOAD_GLOBAL              1 (a)
                6 CALL_FUNCTION            2
                8 POP_TOP
    
    10          10 LOAD_CONST               1 (1)
                12 STORE_FAST               0 (x)
    
    11          14 LOAD_GLOBAL              0 (print)
                16 LOAD_FAST                0 (x)
                18 LOAD_GLOBAL              1 (a)
                20 CALL_FUNCTION            2
                22 POP_TOP
    
    12          24 LOAD_CONST               0 (None)
                26 RETURN_VALUE

区别就在于x在第一个例子中是LOAD_GLOBAL, 而第二个例子中是LOAD_FAST. 原因呢, 是因为:

*If a variable is assigned a value anywhere within the function’s body, it’s assumed to be a local unless explicitly declared as global.*

关键在于上面句子中的 If a variable is **assigned** a value anywhere within the function’s body, it’s **assumed** to be a local unless explicitly declared as global.这句话中的 **If a variable is assigned**, 是assigned的时候

才会当(assumed)成局部变量. 所以我们在函数中任何地方, 一旦有修改x的操作, 那么解释器都会把x当做局部变量来对待的.

*Though a bit surprising at first, a moment’s consideration explains this. On one hand, requiring global for assigned variables provides a bar against unintended side-effects.

On the other hand, if global was required for all global references, you’d be using global all the time. You’d have to declare as global every reference to a built-in function or to a component of an imported module.

This clutter would defeat the usefulness of the global declaration for identifying side-effects.*

上面的解释的意思就是:

1. 如果修改也是默认全局的话, 会有很多副作用(这个想想也知道).

2. 如果全局变量访问的时候需要加上global关键字的话, 那么可能会出现很多很多global关键字.

3. 所以啦, 访问默认是全局的, 然后一旦有修改操作, 那么就是局部的, 必须先定义!!

但是, 如果我们在函数里面修改字典, 但是并没有报错:

.. code-block:: python

    a = {'a': 1}
    
    
    def test():
        print(a)
        a['b'] = 2
        print(a)
        return

字节码中依然是LOAD_GLOBAL, 所以依然把a当成了全局遍历, 所以也就是证明了可变对象的修改并不是assign, 而是modify in-place.

所以可以这么理解, 在函数里面, 修改字典的时候, 是modify in place, 不是reassign, rebinding, 所以解释器会直接根据LEGB原则加载到全局的字典, 然后修改.

而其他不可变对象就不行, 不可变对象在函数里面若有修改操作, 也就是reassign, rebinding操作, 解释器就把它当成局部变量, 因为test中只有打印语句就报没有定义的错误, 而test1中有reassign, 则

解释器就把x当做局部遍历, 当你要操作一个局部变量的时候，必须先赋值

有时候得抠字眼一下才能理解.

先赋值
=============

其实python中是遵循先定义, 再使用的原则的

.. code-block:: python

    def test():
        print(x)
    
    x = 1
    
    test()

看起来print(x)是在x赋值之前就调用了，应该报未赋值的异常，但是其实是可以正常运行的

这是因为先赋值是指 **操作之前** 必须赋值, 当解释器执行到test函数定义的时候，是执行了函数定义操作，但是并没有操作x, 然后我们赋值x, 然后调用test, test中操作了x

所以这个情况就是操作之前赋值了

如果把x的赋值移到test调用之后, 就会报未赋值异常了


