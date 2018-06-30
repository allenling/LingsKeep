###########################
curio中的sleep和timeout实现
###########################

关键在于sleepq这个队列, sleepq是TimeQueue类的实例, 是使用绝对流逝时间来判断timeout和sleep, 支持push和pop操作

然后kernel在主while True循环里面优先查看是否有active或者ready的task, 有则执行

否则进入休眠判断, 休眠是调用select, 其中select的超时则是通过sleepq计算而来

所以也就是从休眠队列中, 获取最小休眠时间, 然后如果中间被唤醒, 也就是select被唤醒, 那么优先处理被唤醒的event

否则进入timeout判断

例子
======

.. code-block:: python
    
    import curio
    
    
    async def timeout_func():
        await curio.sleep(10)
        return 'done'
    
    
    async def test_timeout():
        try:
            res = await curio.timeout_after(1, timeout_func)
        except curio.TaskTimeout:
            print('timeout!!!')
        else:
            print(res)
        return
    
    
    def main():
        curio.run(test_timeout)
        return
    
    
    if __name__ == '__main__':
        main()

看看timeout_after是什么:

.. code-block:: python

    def timeout_after(seconds, coro=None, *args):
        '''
        Raise a TaskTimeout exception in the calling task after seconds
        have elapsed.  This function may be used in two ways. You can
        apply it to the execution of a single coroutine:
    
             await timeout_after(seconds, coro(args))
    
        or you can use it as an asynchronous context manager to apply
        a timeout to a block of statements:
    
             async with timeout_after(seconds):
                 await coro1(args)
                 await coro2(args)
                 ...
        '''
        if coro is None:
            return _TimeoutAfter(seconds, False)
        else:
            return _timeout_after_func(seconds, False, coro, args)

而_timeout_after_func则也是调用_TimeoutAfter

.. code-block:: python

    async def _timeout_after_func(clock, absolute, coro, args, ignore=False, timeout_result=None):
        coro = meta.instantiate_coroutine(coro, *args)
        async with _TimeoutAfter(clock, absolute, ignore=ignore, timeout_result=timeout_result):
            return await coro


所以, 主要代码就是_TimeoutAfter, 也就是每次你调用timeout_after, 那么都会生成一个_TimeoutAfter实例

也就是每个_TimeoutAfter都用来表示一个task中timeout的流程


_TimeoutAfter
==================

这个类主要思路是:

1. timeout是通过流逝的时间, 而不是通过timeout计算出绝对时间, 来计算

2. 执行传入的coro的时候, 然后把coro加入到kernel中的sleepq队列中


看看执行async with语法的时候的流程:


.. code-block:: python

    class _TimeoutAfter(object):
    
        async def __aenter__(self):
            # 拿到当前kernel正在执行的task
            task = await current_task()
            if not self._absolute and self._clock:
                # 这里, _clock是获取当前已经流逝时间
                self._clock += await _clock()
                self._absolute = False
            self._deadlines = task._deadlines
            self._deadlines.append(self._clock)
            self._prior = await _set_timeout(self._clock)
            return self


1. current_task是一个系统调用, 获取当前运行的task, 在例子中, 这里

   拿到的就是 *test_timeout* 这个函数

2. 然后self._absolute是False, 因为我们不是用绝对时间去计算timeout的, 所以接下来我们

   通过_clock系统调用拿到当前已经流逝的时间, _clock = time.monotonic

   所以, self._clock += _clock就是获取我们timeout的下一个流逝的目标时间

   (这个说法有点绕, 后面就理解了)

3. 后面的self._deadlines操作就是把目标流逝时间加入到_deadlines列表中


4. _set_timeout是一个系统调用, 那么也就是说我们会把我们目标流逝时间

   计入到kernel中的某个队列中!!!!!!


_set_timeout
================

_set_timeout的系统调用是

.. code-block:: python

    def _set_timeout(clock):
        '''
        Set a timeout for the current task that occurs at the specified clock value.
        Setting a clock of None clears any previous timeout.
        '''
        return (yield (_trap_set_timeout, clock))


所以, 找到trap对应的函数是_trap_set_timeout, 参数是clock, 也就是流逝的绝对时间


.. code-block:: python

    class Kernel:

        def _trap_set_timeout(timeout):
            # timeout是流逝的绝对时间
            old_timeout = current.timeout
            if timeout is None:
                # If no timeout period is given, leave the current timeout in effect
                pass
            else:
                _set_timeout(timeout)
                if old_timeout and current.timeout > old_timeout:
                    current.timeout = old_timeout

            current.next_value = old_timeout


然后, 我们进入到_set_timeout函数

.. code-block:: python

    class Kernel:

        def _set_timeout(clock, sleep_type='timeout'):
            if clock is None:
                sleepq.cancel((current.id, sleep_type), getattr(current, sleep_type))
            else:
                sleepq.push((current.id, sleep_type), clock)
            setattr(current, sleep_type, clock)


所以, 也就是把流逝的绝对时间加入到sleepq这个队列中


sleepq
=========

sleepq是一个TimeQueue类, 这个类主要的作用就是:

1. push, 接收一个绝对流逝的时间, 然后把绝对流逝时间加入到heapq中

   而heapq就是一个优先级队列, 所以, push就是说保存了一个优先级队列

2. pop, 每次从heapq中pop出一个最小的流逝时间, 判断该流逝时间是否大于当前时间


而kernel会判断最小流逝时间是否大于当前时间, 如果是, 则表示有task过期了

否则不做处理

.. code-block:: python

    class TimeQueue:

        def __init__(self, timeslice=1.0):
            self.near_deadline = 0.0
            self.timeslice = timeslice
            self.near = []

            # 下面是bucket的说明, 把小于4s过期放入0号bucket, 其他以此类推
            # Set of buckets for timeouts occurring 4, 16, 64s, 256s, etc. in the future (from deadline)
            self.far = [ {} for _ in range(8) ]
            self.far_deadlines = [self.near_deadline] + [self.near_deadline + 4 ** n for n in range(1,8) ]
    
        def push(self, item, expires):
            '''
            Push a new item onto the time queue.
            '''
            if expires is None:
                return
    
            # If the expiration time is closer than the current near deadline,
            # it gets pushed onto a heap in order to preserve order
            if expires < self.near_deadline:
                heapq.heappush(self.near, (expires, item))
    
    
            # Otherwise, the item gets dropped into a bucket for future processing
            else:
                delta = expires - self.near_deadline
                bucketno = 0 if delta < 4.0 else int(0.5*log2(delta))
                if bucketno > 7:
                    bucketno = 7
                self.far[bucketno][item] = expires

其中, self.near_deadline是缓存的, 最近一个过期时间, 所以:

1. 如果传入的expires小于self.near_deadline, 加入到优先队列中(heapq)

2. 否则则把expires加入到bucket中, bucket的概念是这样的

   把4s内过期的item归到0号bucket中, 然后其他以此类推

3. 最后, _set_timeout则设置current的timeout属性, 此时current是test_timeout, 而不是timeout_func

   *setattr(current, sleep_type, clock)*, 其中sleep_type='timeout', clock是绝对流逝时间


经过上面的流程, _trap_set_timeout这个系统调用的主要流程就结束了, 也就是with _TimeoutAfter中

执行__aenter__函数已经执行完了, 所以接下执行with中的代码块, 也就是我们的coro, 也就是例子中的timeout_func

.. code-block:: python

    async def _timeout_after_func(clock, absolute, coro, args, ignore=False, timeout_result=None):
        coro = meta.instantiate_coroutine(coro, *args)
        async with _TimeoutAfter(clock, absolute, ignore=ignore, timeout_result=timeout_result):
            # 接下来是执行这里!!!!!!!!!!!!!!!!!!
            return await coro

在timeout_func中, 也就是执行await sleep, 也就是进入休眠.


sleep系统调用
===============

.. code-block:: python

    async def sleep(seconds):
        '''
        Sleep for a specified number of seconds.  Sleeping for 0 seconds
        makes a task immediately switch to the next ready task (if any).
        '''
        return await _sleep(seconds, False)


在kernel中, _sleep函数

.. code-block:: python

    class Kernel:

        def _trap_sleep(clock, absolute):
            # 检查是否应该被取消
            # 如果被取消了, 则直接退出
            if _check_cancellation():
                return

            # We used to have a special case where sleep periods <= 0 would
            # simply reschedule the task to the end of the ready queue without
            # actually putting it on the sleep queue first. But this meant
            # that if a task looped while calling sleep(0), it would allow
            # other *ready* tasks to run, but block ever checking for I/O or
            # timeouts, so sleeping tasks would never wake up. That's not what
            # we want; sleep(0) should mean "please give other stuff a chance
            # to run". So now we always go through the whole sleep machinery.
            # 如果不是绝对时间, 那么计算绝对流逝时间
            if not absolute:
                clock += time_monotonic()
            # 然后再次进入_set_timeout流程
            # 也就是走一遍TimeQueue的push
            _set_timeout(clock, 'sleep')
            # 把task加入到suspend队列
            # 其中cancel的回调函数是lambda, 也就是setattr, 也就是设置
            # task的sleep属性为None!!!!!!!
            _suspend_task('TIME_SLEEP', 
                          lambda task=current: setattr(task, 'sleep', None))

所以, 经过上面的流程之后, sleepq这个对象中, far(也就是bucket)为:

.. code-block:: python

    [{}, {}, {}, {}, {}, {(2, 'timeout'): 453547.251985775}, {(2, 'sleep'): 457571.131513948}, {}]

所以, 也就是id=2的task, 也就是test_timeout存在于两个bucket中, 一个是自己的调用的timeout, 一个是timeout_func的sleep

此时current, 也就是Task(id=2)中:

.. code-block:: python
   
   current.timeout = 453547.251985775

   current.sleep   = 457571.131513948

_suspend_task
================

把task加入到暂停队列(suspend queue)

.. code-block:: python

    def _suspend_task(state, cancel_func):
        nonlocal current
        current.state = state
        current.cancel_func = cancel_func
        
        # Unregister previous I/O request. Discussion follows:
        #
        # When a task performs I/O, it registers itself with the underlying
        # I/O selector.  When the task is reawakened, it unregisters itself
        # and prepares to run.  However, in many network applications, the
        # task will perform a small amount of work and then go to sleep on
        # exactly the same I/O resource that it was waiting on before. For
        # example, a client handling task in a server will often spend most
        # of its time waiting for incoming data on a single socket.
        #
        # Instead of always unregistering the task from the selector, we
        # can defer the unregistration process until after the task goes
        # back to sleep again.  If it happens to be sleeping on the same
        # resource as before, there's no need to unregister it--it will
        # still be registered from the last I/O operation.
        #
        # The code here performs the unregister step for a task that
        # ran, but is now sleeping for a *different* reason than repeating the
        # prior I/O operation.  There is coordination with code in _trap_io().
    
        if current._last_io:
            _unregister_event(*current._last_io)
            current._last_io = None
    
        current = None


所以, 也就是:

1. 设置current的状态是传入的状态, 这里是TIME_SLEEP

2. 设置cancel函数, 这里就是传入的 *lambda task=current: setattr(task, 'sleep', None)*

3. 把当前正在执行的task设置为None, 所以kernel会去获取下一个任务来执行

kernel检查timeout
=====================

经过上面的流程之后, kernel中的休眠队列(sleepq)中就保存了Task(id=2), 激活队列(active)和就绪队列(ready)则为空

接着, kernel继续while True循环, kernel会计算自己需要休眠多久, 然后被唤醒之后, 校验超时!!!!

.. code-block:: python


    class Kernel:
    
        while True:
    
           # ------------------------------------------------------------
           # I/O Polling/Waiting
           # ------------------------------------------------------------
    
           if ready or not main_task:
               timeout = 0
           else:
               current_time = time.monotonic()
               timeout = sleepq.next_deadline(current_time)
           try:
               events = selector_select(timeout)
           except OSError as e:     # pragma: no cover
               # If there is nothing to select, windows throws an
               # OSError, so just set events to an empty list.
               if e.errno != getattr(errno, 'WSAEINVAL', None):
                   raise
               events = []

           # select被唤醒, 有可能是有IO到来, 所以判断一下
           # 这里例子中是没有, 所以流程先过掉
           for key, mask in events:
               pass

           # 接下来进入到timeout的判断
           # ------------------------------------------------------------
           # Time handling (sleep/timeouts
           # ------------------------------------------------------------

           # 计算当前流逝时间
           current_time = time_monotonic()

           for tm, (taskid, sleep_type) in sleepq.expired(current_time):
               # 接着判断timeout, 我们后面将讲
               # 这里先略过, 只讲主要流程
               pass




判断ready已经main_task, 在例子中, ready是空list, 而main_task则是Task(id=2), 也就是我们的test_timeout函数

然我们进入else代码块:

1. 计算current_time, 也就是当前流逝的时间

2. 从sleepq中拿到下一个需要被唤醒的时间, 也就是sleepq.next_deadline

   传入current_time是说比对当前流逝时间和sleepq中保存的流逝时间做对比

   取最小一个


3. 进入select休眠, timeout则是2中计算出来的

4. 然后进入timeout判断, 也就是for一下sleepq.expired


for sleepq.expired
========================

这里, 拿到sleepq中, 最小timeout的流逝时间, 然后expired返回的都是超时的task

所以, 如果需要cancel, 则cancel, 然后接着设置task.next_exc是TaskTimeout, 然后重新调度task

也就是把task加入到ready队列中, 那么我们执行task的时候, 发现其next_exec不是None, 则throw


.. code-block:: python

    # ------------------------------------------------------------
    # Time handling (sleep/timeouts
    # ------------------------------------------------------------

    current_time = time_monotonic()
    for tm, (taskid, sleep_type) in sleepq.expired(current_time):
        # When a task wakes, verify that the timeout value matches that stored
        # on the task. If it differs, it means that the task completed its
        # operation, was cancelled, or is no longer concerned with this
        # sleep operation.  In that case, we do nothing
        if taskid in tasks:
            task = tasks[taskid]
            if sleep_type == 'sleep':
                # 如果task超时是因为sleep
                # 说明task应该被唤醒了, 那么重新调度task
                # 保存被唤醒的时间, 也就是被唤醒时候绝对流逝时间current_time
                # 将会被传递给sleep的callback
                if tm == task.sleep:
                    task.sleep = None
                    _reschedule_task(task, value=current_time)
            else:
                # 如果task休眠是因为timeout
                # 并且tm == task.timeout, 也就是task大到了超时时间了, 那么
                # 如果task是休眠状态, 执行cancel, 然后重新调度
                # 如果task是running状态, 则把task设置为等待cancel, 原因是TaskTimeout
                if tm == task.timeout:
                    task.timeout = None
                    # If cancellation is allowed and the task is blocked, reschedule it
                    if task.allow_cancel and task.cancel_func:
                        task.cancel_func()
                        _reschedule_task(task, exc=TaskTimeout(current_time))
                    else:
                        # Task is on the ready queue or can't be cancelled right now,
                        # mark it as pending cancellation
                        task.cancel_pending = TaskTimeout(current_time)




所以:

1. 如果是sleep, 那么重新调度

2. 如果是timeout, 那么如果task是休眠状态, 则执行cancel, 然后重新调用task, 设置task的异常是TaskTimeout
   
   如果task是running状态, 那么不能马上cancel, 需要把task标记为等待cancel(cancel_pending)状态
   
   那么下一次task被执行的是, 就会执行cancel了


没有超时怎么办?
====================

上面的流程就是task已经超时了, 那么没有超时呢?


我们把例子中, timeout_func中sleep改为1, 然后timeout_func中timeout_after传入10s



