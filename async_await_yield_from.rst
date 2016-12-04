
asyncio.coroutine中用yield而不是yield from
==============================================

这样在asyncio的evnt loop中会检查并报错


types.coroutine/yield from vs async/await
===========================================

Awaitable对象
-----------------

定义了__await__方法的对象， 并且__await__返回一个可迭代对象(定义了__iter__)，不能是一个协程。比如asyncio.Future

.. code-block:: python

    class Future:
    
    
        def __iter__(self):
            if not self.done():
                self._blocking = True
                yield self  # This tells Task to wait for completion.
            assert self.done(), "yield from wasn't used with future"
            return self.result()  # May raise too.
    
        if compat.PY35:
            __await__ = __iter__ # make compatible with 'await' expression

这里__await__返回了自己， 因为自己也是一个可迭代对象(定义了__iter__).


相同点
--------

基本上, await和yield from功能差不多, 都是暂停， 然后返回一个awaitable对象(协程也是awaitable对象).

基本上可以说await == yield from. 只是await更严格， 后面只能跟awaitable对象.

async 定义的函数是一个Collections.abc.Coroutine实例


opcode的区别
----------------

types.coroutine的装饰器装饰一个带有yield from语句的函数的时候， yield from接受一个生成器或者协程或者awaitable对象.

types.coroutine的作用是将CO_ITERABLE_COROUTINE的code flag加入到func的codeobject中(也就是func.__code__), 然后VM在看到

CO_ITERABLE_COROUTINE的flag， 就调用GET_YIELD_FROM_ITER， GET_YIELD_FROM_ITER检查yield from是否跟着参数arg是否是一个生成器或者协程， 如果不是， 则调用iter(arg)


而await后面接的是awaitable对象的时候， VM的opcode就是GET_AWAITABLE, 由于__await__方法返回一个可迭代对象， 所以GET_AWAITABLE就是获取可迭代对象.


types.coroutine/yield from和async/await的的目的是严格定义协程， 只有后者定义的才是原生协程， 前面那个定义的是基于生成器的协程, 确保一个生成器的使用目的.

也就是说，使用types.coroutine装饰的生成器的目的就是用来作为协程. 由于Python不是静态编译语言， 所以在使用types.coroutine装饰的时候， Python不知道该生成器是用来作为协程的，
所以要在opcode级别，也就是VM级别上加以区分.



只有基于生成器的协程才能暂停，调用send发送数据回程序以便重新启动。比如asyncio.sleep

.. code-block:: python

    In [59]: x=asyncio.sleep(1)
    
    In [60]: q=x.send(None)
    
    In [61]: q
    Out[61]: <Future pending>
    
    In [62]: q.done()
    Out[62]: False
    
    In [63]: q.set_result({'a':1})
    
    In [64]: q.done()
    Out[64]: True
    
    In [65]: q
    Out[65]: <Future finished result={'a': 1}>


 asyncio.sleep启动之后， 返回一个Future对象，暂停，然后event loop就是对这个Future检查，检查其是否已经执行完io了(查看Future是否完成， 调用Future.done())

 一般调用Future.set_result就能把Future的状态变成完成(Future.done()==True)


async/await是API
-------------------

也就是说async/await是一种定义，定义一个函数的作用是用来作为协程。而协程的工作流如何实现，是需要自己取实现的，比如asyncio就是一种实现，一种官方实现。

然后curio实现了自己的event loop.


若asyncio.coroutine中yield from不是一个await对象呢
===================================================

.. code-block:: python

  @asyncio.coroutine
  def my_coroutine_two():
      yield from (i for i in range(3))
  
  loop = asyncio.get_event_loop()
  tasks2 = [my_coroutine_two('task21')]
  
  loop.run_until_complete(asyncio.wait(tasks2))
  loop.close()

由之前所说，asyncio.coroutine会将CO_ITERABLE_COROUTINE这样一个flag加入到函数的code object中， 这样
VM的opcode就是GET_YIELD_FROM_ITER， 这样若yield from之后不是一个await对象

在asyncio中的event loop中， 若yield from得到的不是一个


asyncio的真相
=================

至少在linux下，异步的基础是select/epoll.


单进程使用epll， 然后注册多个fd， 每个fd的call back先拿到io数据，然后把数据设置到future中， 也就是future.set_result, 然后send(future)回对应的协程。

就是这样！没什么很神秘的。

因为epoll可以监听的fd理论上是无限多个，并且每次io完之后会unregister某个fd，所以单进程下，还是以开很多很多个协程。


之前想到太复杂，以为除了epoll之外，还有其他方法取提醒loop，某个程序需要启动，或者单进程下，程序pending的IO如何同时运行，并且提醒loop。


虽然Python之父在介绍asyncio的slides中声称不喜欢call back， 但是思路还是基于epoll模式的call back的。


简单的sleep异步的思路是，loop负责用heap以每一个程序下一次重新启动时间来排序，最快重新启动的程序排在最后，然后循环pop出该函数

然后查看我们是否到了程序需要重新启动的时间，比如最快重新启动程序时间5秒，也就是一个程序5秒后启动，现在已经过了2秒，然后我们需要再次sleep 3秒，之后启动。

所以，这个例子中，loop是启动下一个程序的时间的。


但是，对于真正的IO，并不知道下一次启动程序的具体时间，怎么办。这就需要有人取提醒loop，某个程序需要重新启动，这个角色就是epoll。



