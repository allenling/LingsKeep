curio
=======

v0.8


引发Exception
================


curio中引发异常是通过把exception发送到task中, 然后等到task被调度的时候, 先检查看是否有异常, 有的话直接引发异常

发送异常是通过生成器/协程的throw关键字来实现的

.. code-block:: python

    # curio.kernel, line: 816
    # ------------------------------------------------------------
    # Run ready tasks
    # ------------------------------------------------------------
    for _ in range(len(ready)):
        current = ready_popleft()
        try:
            with TaskExecutor(task=current) as executor:
                # The current task runs traps until it suspends
                while current:
                    # 这里next_exc就是current是否需要发送异常, 比如CancelError
                    if current.next_exc is None:
                        trap = current._send(current.next_value)
                        current.next_value = None
                    else:
                        # 如果有异常, throw一下
                        trap = current._throw(current.next_exc)
                        current.next_exc = None


这个通过已发异常来终止运行的方法, 在线程中也经常用, 线程中是用ctype.pythonapi来设置线程异常, 比如dramatiq就是这样处理超时的


异步和同步
============

http://curio.readthedocs.io/en/latest/devel.html#programming-with-threads

一些同步操作, 大多数是系统调用, 比如socket.create_connection这种, 异步代码最好不要直接执行

然后curio的做法是把这个操作放到一个线程中, 比如curio.open_connection, open_connection最终调用到curio.socket.create_connection:

.. code-block:: python

    @wraps(_socket.create_connection)
    async def create_connection(*args, **kwargs):
        # 线程中执行
        sock = await workers.run_in_thread(_socket.create_connection, *args, **kwargs)
        return io.Socket(sock)

这样也给了一些针对现在一些同步库进行协程化的思路, 比如requests的协程化, 针对系统调用如果异步, 可以就使用线程


spawn and (must)join
========================

我貌似记得0.8之前的版本是可以一旦spawn(coro), 就会至少执行到coro的第一个yield点的, 因为在 `文章 <https://vorpus.org/blog/some-thoughts-on-asynchronous-api-design-in-a-post-asyncawait-world/>`_

中提到curio的spawn是causality的, 也就是会执行到coro的yield点才会继续. 但是0.8的curio好像不行了.

curio有个小细节, 就是你spawn的话, 不会执行spawn里面的协程序, 如果你想spawn一个协程的时候运行直到它挂起(yield)

那么可以加一层协程, 然后join, 例如:

一直spawn
------------

.. code-block:: python

    async def mycoro():
        await curio.sleep(10)
        return
    
    async main():
        for _ in range(10):
            task = await curio.spawn(mycoro)
        return

这样的话会一直spawn, 直到for结束, spawn的时候会把task放入到ready队列, 并不会执行mycoro, 如果你要执行的话, 只能调用

task.join, 那么这样的话for就是阻塞在tasjk.join上了

并且这样有个问题, 如果task超过一定数量的话, 那么cpu会吃满

wrap spawn
------------

wrap spawn的思路是在coro前面加一层coro, 称为wrap_coro，然后wrap.join一下, 这样我们的spawn的时候能够执行到coro第一个await

启动

.. code-block:: python

    async def mycoro():
        await curio.sleep(10)
        return

    async def wrap_spawn():
        c = await curio.spawn(mycoro)
        return

    async main():
        for _ in range(10):
            task = await curio.spawn(wrap_spawn)
            task.join()
        return

我们join那个wrap_spawn任务的话会执行到mycoro的第一个await, 然后退出, 这样就达到了我们一边spawn, 并且能同时执行mycoro, 并且不会阻塞在join上



task group and cancel and timeout_after
==========================================


timeoutafter这个函数结束之后, 会引起一个CancelError的异常, 然后TaskGroup收到这个异常的话, 就会把剩余还在执行的task都cancel掉:

注意的是, TaskTimeout这个异常是继承自CancelError的, 所以意思就是遇到了TaskTimeout, 也就是说被cancel, 遇到了CancelError

TaskGroup.join:
------------------

.. code-block:: python

        while self._running:
            try:
                await self._sema.acquire()
            except CancelledError:
                # 这里会遇到kernel引发的tasktimeouterror(由timeout_after设置的时钟)
                # 然后tasktimeout这个异常继承自cancelerror, 所以这里就cancel了
                # If we got cancelled ourselves, we cancel everything remaining.
                # Must bail out by re-raising the CancelledError exception (oh well)
                for task in list(self._running):
                    # 依次去cancel掉还在执行的task
                    await task.cancel()
                    task._taskgroup = None
                raise


async thread
==============

详细看: https://forum.dabeaz.com/t/what-is-asyncthread-target-for/273

然后其是这样的, 如果一个计算任务, 你需要await结果的话, 调用:

.. code-block:: python

    import curio
    
    
    def syn_thread_func(a, b):
        return sum([a, b])
    
    
    async def test_async_thread():
        athread = curio.thread.AsyncThread(target=syn_thread_func, args=(1, 2))
        await athread.start()
        data = await athread.join()
        print('done, data is: %s' % data)
        return

然后如果你要调用curio.thread.AWAIT, `例子 <http://curio.readthedocs.io/en/latest/reference.html#AWAIT>`_, 的话, 必须保证curio.thread.AsyncThread.target这个函数

没有执行完成, 不然它会造成线程里面的异常任务退出, 会挂起

