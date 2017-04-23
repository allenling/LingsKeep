await/yield from的区别
=========================

from https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/

1. types.coroutine/asyncio.coroutine
----------------------------------------

在3.5之前，根据PEP342的说明，通过为生成器加入send, throw, close方法来支持协程，也就是把生成器当做协程来看待，也就是所谓的基于生成器的协程.

所以，生成器和协程在一定程度上了同义的。

然后asyncio出现，要区别一个生成器到底是一个平凡的生成器(迭代用)，还是当做协程来使用(暂停重启)，就有了types.coroutine和asyncio.coroutine装饰器用来标识一个
函数是不是用来当做协程作用的。

这两个装饰器是将CO_ITERABLE_COROUTINE这个flag加入到func.__code__.co_flags中, 下面是types.coroutine的源码

0x180 == CO_COROUTINE | CO_ITERABLE_COROUTINE
0x100 == CO_ITERABLE_COROUTINE
Ox80 == CO_COROUTINE == 128


.. code-block:: python

    def coroutine(func):
        """Convert regular generator function to a coroutine."""
    
        if not callable(func):
            raise TypeError('types.coroutine() expects a callable')
    
        if (func.__class__ is FunctionType and
            getattr(func, '__code__', None).__class__ is CodeType):
    
            co_flags = func.__code__.co_flags
    
            # Check if 'func' is a coroutine function.
            # (0x180 == CO_COROUTINE | CO_ITERABLE_COROUTINE)
            # 这里是协程，就返回
            if co_flags & 0x180:
                return func
    
            # Check if 'func' is a generator function.
            # (0x20 == CO_GENERATOR)
            if co_flags & 0x20:
                # TODO: Implement this in C.
                co = func.__code__
                func.__code__ = CodeType(
                    co.co_argcount, co.co_kwonlyargcount, co.co_nlocals,
                    co.co_stacksize,
                    # 这里加入CO_ITERABLE_COROUTINE到CodeType中
                    co.co_flags | 0x100,  # 0x100 == CO_ITERABLE_COROUTINE
                    co.co_code,
                    co.co_consts, co.co_names, co.co_varnames, co.co_filename,
                    co.co_name, co.co_firstlineno, co.co_lnotab, co.co_freevars,
                    co.co_cellvars)
                return func



2. async/await
----------------

3.5之后有了async/await关键字，用来声明协程，让协程和生成器含义区分开。通过async声明的函数就是协程，并且await只能在async定义中出现。


3. awaitable对象
-------------------

awaitable对象是实现了__await__方法的对象，协程是awaitable对象，特别的，基于生成器的协程也是awaitable对象，用inspect.isawatable来检查


4. code_flag的区别
-------------------

使用types.coroutine/asyncio.coroutine定义一个协程

.. code-block:: python

    # This also works in Python 3.5.
    # py34_coro.__code__.co_flags & 0x180 == 256 == 0x100 == CO_ITERABLE_COROUTINE, 标识位基于生成器的协程
    @asyncio.coroutine
    def py34_coro():  
        yield from stuff()

使用async/await定义协程

.. code-block:: python

    # py35_coro.__code__.co_flags & 0x180 == 128 == 0x80 == CO_COROUTINE，标识为协程
    async def py35_coro():  
        await stuff()


5. opcode的区别
--------------------

用dis.dis处理4中出现的两个协程

.. code-block:: python

    >>> dis.dis(py34_coro)
      2           0 LOAD_GLOBAL              0 (stuff)
                  3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
                  6 GET_YIELD_FROM_ITER
                  7 LOAD_CONST               0 (None)
                 10 YIELD_FROM
                 11 POP_TOP
                 12 LOAD_CONST               0 (None)
                 15 RETURN_VALUE

-- code-block:: python

    >>> dis.dis(py35_coro)
      1           0 LOAD_GLOBAL              0 (stuff)
                  3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
                  6 GET_AWAITABLE
                  7 LOAD_CONST               0 (None)
                 10 YIELD_FROM
                 11 POP_TOP
                 12 LOAD_CONST               0 (None)
                 15 RETURN_VALUE


区别在于GET_YIELD_FROM_ITER和GET_AWAITABLE两个opcode上


5.1 GET_YIELD_FROM_ITER
++++++++++++++++++++++++++++++

GET_YIELD_FROM_ITER是先检查其参数是不是一个协程或者生成器，如果不是，对参数调用iter来获取它的迭代器对象。

并且yield from后面跟协程对象，必须是yield from出现在一个用types.coroutine/asyncio.coroutine装饰的函数上面，比如:

.. code-block:: python

    async def coro():
        await asyncio.sleep(1)
    
    # 在一个不用types.coroutine/asyncio.coroutine装饰的协程中，yield from一个协程
    def test():
        yield from coro()

    t = test()
    # 这里报错: TypeError: cannot 'yield from' a coroutine object in a non-coroutine generator
    t.send(None)

    # 加上装饰器就可以了
    @types.coroutine
    def test():
        yield from coro()

5.2 GET_AWAITABLE
+++++++++++++++++++++

GET_AWAITABLE限制就是只能跟awaitable对象, 也就是async定义的协程或者用types.coroutine/asyncio.coroutine装饰的协程



6. 最后
---------
You may be wondering why the difference between what an async-based coroutine and a generator-based coroutine will accept in their respective pausing expressions? The key reason for this is to make sure you don't mess up and accidentally mix and match objects that just happen to have the same API to the best of Python's abilities. Since generators inherently implement the API for coroutines then it would be easy to accidentally use a generator when you actually expected to be using a coroutine. And since not all generators are written to be used in a coroutine-based control flow, you need to avoid accidentally using a generator incorrectly. But since Python is not statically compiled, the best the language can offer is runtime checks when using a generator-defined coroutine. This means that when types.coroutine is used, Python's compiler can't tell if a generator is going to be used as a coroutine or just a plain generator (remember, just because the syntax says types.coroutine that doesn't mean someone hasn't earlier done types = spam earlier), and thus different opcodes that have different restrictions are emitted by the compiler based on the knowledge it has at the time.


简单来说: 定义协程上，yield from和async/await目的希望你们不要被平凡的生成器和基于协程的生成器混淆，因为两者都有一样的API(send,throw,close)，又因为Python不是静态编译语言，所以要在定义上要区别开来。

