debug python步骤
==================

https://devguide.python.org/setup/

cpython/Misc/SpecialBuilds.txt

http://pythonextensionpatterns.readthedocs.io/en/latest/debugging/debug_python.html

用gdb调试cpython: http://kharkivpy.org.ua/wp-content/uploads/2017/01/debugging-with-gdb.pdf

vscode调试cpython


gil的实现
=============

新gil基于时间间隔的强制释放锁

3.2之前的基于(100)tick的切换

由于python的线程都是lwp, 所以是可以并发的，只是没拿到gil的线程是不能运行


.. raw:: 

    thread1 -----running---> release gil　-----wait gil---->
    
    thread2 -----wait gil--> got     gil  ------running----->
    
    thread3 -----wait gil--> still  wait  ------wait gil---->


**以下都是David Beazley的文章**

gil运行模式
==============================================

http://www.dabeaz.com/python/GIL.pdf

老的GIL，CPU密集型是100个ticks去释放一次GIL，I/O密集型则是执行IO操作的时候释放GIL

比较
-----

py3.2之前 GIL，两个cpu密集型的线程，两核比单核慢很多。

有时候ctrl+c不能杀死进程
---------------------------
 
是因为如果解释器接收到信号之后


1. The reason Ctrl-C doesn't work with threaded programs is that the main thread is often blocked on an uninterruptible thread-join or lock.

2. Since it's blocked, it never gets scheduled to run any kind of signal handler for it

3. And as an extra little bonus, the interpreter is left in a state where it tries to thread-switch after every tick (so not only can you not interrupt your program, it runs slow as hell!)

1. 是每一个tick就释放gil让其他线程执行, 直到当前执行的线程为主线程为止.

2. 若主线程被不可中断的thread join或者lock给阻塞了，然后主线程就不会被OS给唤醒，也就是不会重新启动.

3. 这个时候程序由于check很频繁, 导致程序呢很慢


两个cpu密集型thread
----------------------------

多核情况下慢的原因是释放GIL之后的信号处理上

1. GIL thread signaling is the source of that

2. After every 100 ticks, the interpreter

3. Locks a mutex

4. Signals on a condition variable/semaphore where another thread is always waiting

5. Because another thread is waiting, extra pthreads processing and system calls get triggered to deliver the signal

文章接下来的图示表示了, 两个cpu密集型任务, 一旦使用了线程, 系统调用会大大增加.

osx上:

1. 单核情况下顺序执行两个任务只有736个unix系统调用和117个Mach系统调用, 如果用两个线程, 居然有3百多万个Mach的系统调用.
   
2. 多核的时候, 多线下unix系统调用没什么变化, 但是Mach系统调用能增加到9百万个.


io和cpu密集型竞争
---------------------

1. 线程切换还依赖于OS的切换，在一般OS中，cpu密集型线程是低优先级，而IO密集型线程是高优先级

2. CPU竞态，也就是多核下，两个线程(CPU密集型)同时运行，然后第一个释放掉GIL之后，在信号发给另外一个线程的时候，第一个线程有获取了GIL，这是因为第一个线程还在第一个核上运行，

3. 而第二个线程就一直获取失败，这样另一个线程就过很久才能拿到GIL(参考之后的convoy effect)

4. 一个cpu密集的线程和一个IO密集的线程分别在不同核心上运行，然后，跟上面一个情况一样，一旦cpu密集型线程拿到GIL，另外一个线程几乎很难拿到GIL


新gil调度图解
================

http://www.dabeaz.com/python/NewGIL.pdf

Py3.2之后GIL被重写了，cpu密集型的线程释放GIL不再是基于tick数目了, 而是基于 **请求释放的机制**:

1. 线程a会一直执行, 没有有其他线程发起drop gil的请求, 一直执行.

2. 线程b开始执行, b一开始检查有其他线程拿到了gil, 那么会默认等待5ms

3. 5ms期间a释放了gil, 比如a进入io的时候, 就去抢gil

4. 5ms超时了其他线程依然没有释放gil, 那么b发起一个drop gil请求, 然后等待

5. a收到drop gil的请求, 释放gil, 然后重新去抢gil.

6. b得到gil被释放的通知之后, 去抢gil.

其他情况
----------

drop请求频率
---------------

线程a, b, c中, 如果a拿着gil, 让b发起一次drop gil请求之后, c应该不需要再发drop gil请求了, 如果c再发一次drop gil请求, 那么b拿到gil之后, 又要释放了, 结果就是

没人拿到gil!!!

所以说, 在c等待gil释放期间, 判断有人已经发起drop请求了, 就等待就好, 不要发了.

线程调度和gil
----------------

线程a, b, c, a拿着gil, 然后b发起一个drop请求, 那么b, c谁拿到gil?

答案是不一定的, 如果此时os调度了c, 那么就是c拿, 如果先调度b, 那么b拿, 如果b, c同时被调度(多核), 那么抢吧.

是否唤醒是取决os的调度, 也就是线程的优先级的.

详解在: python_gil.rst


新gil存在的问题
================

http://www.dabeaz.com/python/UnderstandingGIL.pdf

新gil的结构: It's a binary semaphore constructed from a pthreads mutex and a condition variable, **持有gil只是用一个真假值表示而已而已**, 但是操作真假值则是受mutx保护的.


新GIL之后依然存在convoy effect。一个cpu密集型线程和一个io密集型线程同时在多核上运行，这样io密集的线程性能将严重下降，原因是，如果io密集型线程进行io操作的时候，会释放掉GIL，然后cpu密集型的线程拿到

GIL，然后在下一个等待超时后将GIL还给io密集型的线程，但是若io密集型的线程的io操作是不需要挂起的呢，比如write数据情况下，由于os有读写缓冲区(假设空间足够)，所以write不会被阻塞，但是线程还是会释放掉

GIL，然后cpu密集型线程就运行了，这样io密集型的线程必须等待下一个等待超时才能获取GIL，这样性能就下降了。

新的GIL消除了gil battle, 但是引入了timeout这样一个时间消耗, 所以对于高负载的io应用来说, gil timeout有可能会影响响应时间.

1. t2中某个fd可读, 然后t2先等待gil timeout, 然后t1是cpu绑定的,
 
   自然不会释放(假设在5毫秒内), 然后t2强制t1释放gil, 然后t1释放, t2拿到gil, 然后运行, 整个过程中t2第一次等待timeout很有可能是失败的，只能强制让t1让出gil

2. 接1的例子, t3是另外一个线程, 然后os唤醒的不是t2而是t3, 那t2只能再继续竞争

3. 接1的例子, 如果t2的io操作可以立即执行完成, 比如发送缓存区的大小大于发送的数据, 则write可以很快完成, 但是io必须释放gil, 所以t1又拿到了, 如果t2有很多io, 但是
  
   大多数都可以几乎立即执行完成的情况下, 释放gil, 再重新获取gil的timeout就变得很多了 

4. A Possible Solution: 
    - If a thread is preempted by a timeout, it is penalized with lowered priority (bad thread)
    - If a thread suspends early, it is rewarded with raised priority (good thread)
    - High priority threads always preempt low priority threads

**python3.6中有个FORCE_SWITCHING的设置, 可以是得a线程释放之后, 一定会等待其他线程拿到gil之后才继续获取gil! 看起来会缓解convoy effect**


护航效应
===========================================

https://bugs.python.org/issue794

convoy effect的issue

os中的convoy effect: http://www.geeksforgeeks.org/convoy-effect-operating-systems/



