gil的实现
=============

新gil基于时间间隔的强制释放锁

3.2之前的基于(100)tick的切换

由于python的线程都是lwp, 所以是可以并发的，只是没拿到gil的线程是不能运行的啦


.. raw:: 
    thread1 -----running---> release gil　-----wait gil---->
    
    thread2 -----wait gil--> got gil      ------running----->
    
    thread3 -----wait gil--> still wait   ------wait gil---->


**以下都是David Beazley的文章**

gil运行模式
==============================================

http://www.dabeaz.com/python/GIL.pdf

老的GIL，CPU密集型是100个ticks去释放一次GIL，I/O密集型则是执行IO操作的时候释放GIL

比较
-----

py3.2之前 GIL，两个cpu密集型的线程，两核比单核慢很多。

问题
-----

有时候ctrl+c不能杀死进程
~~~~~~~~~~~~~~~~~~~~~~~~~~~
 
是因为如果解释器接收到信号之后，是每一个tick就释放gil让其他线程执行, 直到当前执行的线程为主线程为止. 若主线程被不可中断的thread join或者lock给阻塞了，然后主线程就不会被OS给唤醒，也就是不会重新启动.

这个时候程序由于check很频繁，运行就很慢:

1. The reason Ctrl-C doesn't work with threaded programs is that the main thread is often blocked on an uninterruptible thread-join or lock.

2. Since it's blocked, it never gets scheduled to run any kind of signal handler for it

3. And as an extra little bonus, the interpreter is left in a state where it tries to thread-switch after every tick (so not only can you not interrupt your program, it runs slow as hell!)


多核情况下慢的原因是释放GIL之后的信号处理上
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. GIL thread signaling is the source of that

2. After every 100 ticks, the interpreter

3. Locks a mutex

4. Signals on a condition variable/semaphore where another thread is always waiting

5. Because another thread is waiting, extra pthreads processing and system calls get triggered to deliver the signal

竞态
~~~~~~~~


1. 线程切换还依赖于OS的切换，在一般OS中，cpu密集型线程是低优先级，而IO密集型线程是高优先级

2. CPU竞态，也就是多核下，两个线程(CPU密集型)同时运行，然后第一个释放掉GIL之后，在信号发给另外一个线程的时候，第一个线程有获取了GIL，这是因为第一个线程还在第一个核上运行，

3. 而第二个线程就一直获取失败，这样另一个线程就过很久才能拿到GIL(参考之后的convoy effect)

4. 一个cpu密集的线程和一个IO密集的线程分别在不同核心上运行，然后，跟上面一个情况一样，一旦cpu密集型线程拿到GIL，另外一个线程几乎很难拿到GIL


新gil实现
=============================================

http://www.dabeaz.com/python/NewGIL.pdf

Py3.2之后GIL被重写了，cpu密集型的线程释放GIL不再是基于tick数目了，但是IO线程切换还是在陷入IO的时候释放GIL。sys.setcheckinterval不再影响线程切换了，而是这个sys.setswitchinterval函数设置解释器切换线程的间隔，默认是0.005s.


2.1  一个线程会一直运行，直到一个全局变量gil_drop_request被设置为1，线程检查到该变量被设置为1之后，说明有其他线程需要GIL，然后当前线程会主动释放掉GIL。

2.2  线程1一直运行，线程2先等待一段时间，在等待时间内，若线程1还没有主动释放掉GIL(陷入IO什么的)，则线程2设置全局变量gil_drop_request=1，然后再次等待，线程1检查到gil_drop_request=1，则主动释放掉GIL，

     同时发送信号给线程2，然后等待线程2的信号。线程2受信之后，拿到GIL，然后发送信号给线程1，表示线程2已经拿到了GIL。

2.3  全局变量gil_drop_request不能被频繁设置，否则一个线程刚刚拿到GIL，另外一个线程恰好等待时间到了，又把gil_drop_request设置为1，则刚刚拿到GIL的线程又要切换。

     2.3.1 On GIL timeout, a thread only sets gil_drop_request=1 if no thread switches of any kind have occurred in that period

     2.3.2 It's subtle, but if there are a lot of threads competing, gil_drop_request only gets set once per "time interval"

     2.3.3 大概意思是，在一个等待超时段内，若没有线程切换，则可以设置gil_drop_request=1，否则，等到下一个等待超时。

     2.3.4 比如，线程1在运行，线程2之后开始等待，然后线程3在线程2等待之后也开始等待，然后线程2超时，然后设置gil_drop_request=1，然后线程1释放掉GIL，线程2拿到GIL，然后此时

           线程3的等待超时了，这个时候不应该去设置gil_drop_request=1，因为在线程3的等待周期内，发生了一次线程切换，所以只能等待下一个等待超时才能设置gil_drop_request=1

2.4  一个线程释放掉GIL之后，其他线程哪一个拿到GIL是由OS来决定的。比如2.3.4的例子中，线程1释放掉GIL，然后OS唤醒线程3，则线程2只能继续等待了。

新gil存在的问题
=========================================================

http://www.dabeaz.com/python/UnderstandingGIL.pdf

新GIL之后依然存在convoy effect。一个cpu密集型线程和一个io密集型线程同时在多核上运行，这样io密集的线程性能将严重下降，原因是，如果io密集型线程进行io操作的时候，会释放掉GIL，然后cpu密集型的线程拿到

GIL，然后在下一个等待超时后将GIL还给io密集型的线程，但是若io密集型的线程的io操作是不需要挂起的呢，比如write数据情况下，由于os有读写缓冲区(假设空间足够)，所以write不会被阻塞，但是线程还是会释放掉

GIL，然后cpu密集型线程就运行了，这样io密集型的线程必须等待下一个等待超时才能获取GIL，这样性能就下降了。

(一个额外的参考，curio和trio这两个Python异步框架，curio的思想是每一个io操作都会引发yield，而trio中，有些不需要阻塞的io操作，则不会yield, 类比GIL的例子，yield就像
切换线程一样)

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


护航效应
===========================================

https://bugs.python.org/issue794

convoy effect的issue

os中的convoy effect: http://www.geeksforgeeks.org/convoy-effect-operating-systems/



