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



