threading模块中的一些概念
==========================

Lock: 同步锁, aquire将会阻塞直到另一方release

RLock: 可重入锁, 与Lock的区别是: 同一个线程多次aquire一个Lock也会阻塞, 而RLock则不会. 每次aquire, 计数器+1, release的时候, 计数器减一, 直到计数器为0才释放

Condition: 条件锁,

Threading.Queue和multiprocessing.Queue
==========================================

python原生的Threading.Queue是线程级别的, 用于在一个进程中不同的线程, 可以正常的put和get, 但是不同进程中, 就不行.

multiprocessing的Queue是启动一个新线程去, 在新线程中将put的数据写入到pipe中, 关键词: pipe, 线程

pipe也就是重定向而已, 一个进程的输出转成另外一个进程的输入. 每一个进程中0,1,2就是标准输入, 标准输出, 标准错误, 将标准输出重定向到一个fd,os.dup2(fd, 1), 将标准输入重定向到一个fd, os.dup2(fd, 0)

然后一个进程的输除就能变成另外一个进程的输入了.

fork之后的变量的id值是一样的, 但是有区别, 又不是一个变量
==========================================================

比如,在主进程中, 初始化一个列表x=[], 将x传入子进程函数, child(x), 然后主进程将数据append到x, x.append('data'), 在子进程中, x是空的 x == [], 但是在主进程和子进程分别id(x), 得到的值是一致的.

id出来的地址是每一个进程的虚拟内存地址而已, 进程间只有fd是共享的, fork只是复制数据而已.

fork之后只有fd是共享的, 但是各个进程间关闭fd缺不会影响各自, 也就是进程1关闭了socket, 但是在进程2中, 同样的socket并不会受影响.

读取stdin
===========

将进程的stdin重定向到fd之后, 就使用sys.stdin.readline来读取每一行的数据了. sys.stdin是一个file-like对象. 不需要我们pip.read之类的.

也可以使用select/poll/epoll来轮询pipe(毕竟pipe也是一种fd).

sys.stdin.readline类似与这样

.. code-block:: python

    while True:
        input = ""
        c = stdin.read(1)
        while c is not EOF:
            input += c
            c = stdin.read(1)
        for line in input.split('\n'):
            yield line


CLOCK_MONOTONIC, time.monotonic
==================================

python的time模块中的monotonic方法返回的是开机之后流逝的秒数。


CLOCK_MONOTONIC是monotonic time

CLOCK_REALTIME是wall time。

monotonic time字面意思是单调时间，实际上它指的是系统启动以后流逝的时间，这是由变量jiffies来记录的。系统每次启动时jiffies初始化为0，每来一个timer interrupt，jiffies加1，也就是说它代表系统启动后流逝的tick数。jiffies一定是单调递增的，因为时间不够逆嘛！
 
wall time字面意思是挂钟时间，实际上就是指的是现实的时间，这是由变量xtime来记录的。系统每次启动时将CMOS上的RTC时间读入xtime，这个值是"自1970-01-01起经历的秒数、本秒中经历的纳秒数"，每来一个timer interrupt，也需要去更新xtime。

wall time不一定是单调递增的。因为wall time是指现实中的实际时间，如果系统要与网络中某个节点时间同步、或者由系统管理员觉得这个wall time与现实时间不一致，有可能任意的改变这个wall time。最简单的例子是，我们用户可以去任意修改系统时间，这个被修改的时间应该就是wall time，即xtime，它甚至可以被写入RTC而永久保存。一些应用软件可能就是用到了这个wall time，比如以前用vmware workstation，一启动提示试用期已过，但是只要把系统时间调整一下提前一年，再启动就不会有提示了，这很可能就是因为它启动时用gettimeofday去读wall time，然后判断是否过期，只要将wall time改一下，就可以欺骗过去了。


UNIX Domain Socket
=======================

socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1

但是UNIX Domain Socket用于IPC更有效率:不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。

UNIX域套接字与TCP套接字相比较，在同一台主机的传输速度前者是后者的两倍。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的.

UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

使用UNIX Domain Socket的过程和网络socket十分相似，也要先调用socket()创建一个socket文件描述符，address family指定为AF_UNIX，type可以选择SOCK_DGRAM或SOCK_STREAM，protocol参数仍然指定为0即可。

UNIX Domain Socket与网络socket编程最明显的不同在于地址格式不同，用结构体sockaddr_un表示，网络编程的socket地址是IP地址加端口号，而UNIX Domain Socket的地址是一个socket类型的文件在文件系统中的路径，这个socket文件由bind()调用创建，如果调用bind()时该文件已存在，则bind()错误返回。


读写默认都是阻塞的，非阻塞需要在send, recv加上一个socket.MSG_DONTWAIT标志.

服务端:

.. code-block:: python

    server_address = './uds_socket'
    
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    
    sock.bind(server_address)
    
    sock.listen(1)

客户端:

.. code-block:: python

    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    
    try:
        sock.connect(server_address)
    except:
        pass



Unix Domain Socket(UDS)和IPC
-------------------------------

有一个很有用的函数socket.socketpair(), 这里返回可以给父子进程通信的socket， 不需要绑定， 直接使用

master sock1

child sock2

master的写如sock1的时候，数据流向sock2， child只需要从sock2接收数据就行，而child写入sock2， 则数据流向sock1， 这样master也只需要从sock1接收数据就行

master:

.. code-block:: python

    send(sock1, data)
    recv(sock1, size)

child:

.. code-block:: python

    send(sock2, data)
    recv(sock2, size)

.. code-block:: python

    import socket
    import os
    import time
    
    
    def child():
        pass
    
    
    def main():
        s1, s2 = socket.socketpair()
        pid = os.fork()
        if pid == 0:
            print 'in master, %s' % os.getpid()
            s2.close()
            count = 10
            time.sleep(1)
            while count:
                s1.send(str(count))
                print 'master write: %s' % count
                time.sleep(1)
                data = s1.recv(1024)
                print 'master recv: %s' % data
                count -= 1
        else:
            print 'in child, %s' % os.getpid()
            s1.close()
            count = 10
            while count:
                data = s2.recv(1024)
                print 'child recv: %s' % data
                time.sleep(1)
                s2.send(str(count))
                print 'child write: %s' % count
                count -= 1
    
    if __name__ == '__main__':
        main()


socketpair的理解: http://liulixiaoyao.blog.51cto.com/1361095/533469/

socketpair会创建两个描述符，但改描述符不属于任何的实际文件系统，而是网络文件系统，虚拟的．同时内核会将这两个描述符彼此设为自己的peer即对端（这里即解决了如何标识读写端，可以想象，两个描述符互为读写缓冲区，即解决了这个问题）

然后应用相应socket家族里的read/write函数执行读写操作．

有了这个基础，即可明白为什么试用fork产生的两个子进程都不关闭读端的时候会竞争，如上所述，他们共享相同的文件表项，有相同的inode和偏移量，两个进程的操作当然是相互影响的．

IPC中，管道(pipe) VS unix domain socket
-----------------------------------------

UNIX-domain sockets are generally more flexible than named pipes. Some of their advantages are:

- You can use them for more than two processes communicating (eg. a server process with potentially multiple client processes connecting);

- They are bidirectional;

- They support passing kernel-verified UID / GID credentials between processes;

- They support passing file descriptors between processes;

- They support packet and sequenced packet modes.

- To use many of these features, you need to use the send() / recv() family of system calls rather than write() / read().


named pipe and unnamed pipe
============================

根据http://www.cs.fredonia.edu/zubairi/s2k2/csit431/pipes.html

named pip也称为fifo, 通常说的pipe是unnamed pipe

1 可用性 
---------

unnamed pipe只在父进程和其子进程中可用

Each end of the pipe is closed individually using normal close() system call. Pipes are only available the process that creates the pipe and it’s descendants.

named pipe在任意进程中可用

Named pipes are also called FIFO’s (first in first out). They have “names” and exist as special files within a file system. (file type p) They exist until they are removed with rm or unlink() They can be used with unrelated process not just descendants of the pipe creator.

并且, named pipe在fs中是存在的一个具体的文件, 在linux中, 文件类型是p, 你可以控制所属用户, 所属组, 权限等, 就像其他文件一样.

2 创建方式
------------

unnamed pipe是os.pipe创建, named pipe是os.mkfifo(os.mknod)


所以, 若你的server和client使用pipe来通信, 你就不能用unnamed pipe, 因为server和client并不是父子进程的关系. named pipe在这种情况下可以使用, 当然, socket更好点.

3. pipe file
----------------

http://unix.stackexchange.com/questions/10050/proc-pid-fd-x-link-number

ll /proc/pid/fd中会看到

10 -> pipe:[5584722]这样的输出, 也就是该程序的fd 0被重定向到pipe中, pipe的inode数字就是5584722, 在子进程(unnamed pipe)或者其他进程(named pipe)中, 你会看到

同样的输出

10 -> pipe:[5584722]

也就是两个进程使用了fd10(pipe)来通信


4. fifo文档
-----------------

http://www.ece.eng.wayne.edu/~gchen/ece5650/lecture3.pdf


process, thread, LWP
==========================

http://www.thegeekstuff.com/2013/11/linux-process-and-threads

process是运行程序的抽象, 包含了一系列的资源, 而thread则是process的逻辑处理器, 也就是执行操作资源的对象. kernel切换可以是进程也可以是线程切换了,
kernel中thread称为lwp

多个thread共享process的虚拟内存地址(但是要注意thread safe, sync).

kernel thread, user thread的区别: Scheduling can be done at the kernel level or user level, and multitasking can be done preemptively or cooperatively.(来自wiki)

thread有user-thread/kernel-thread两种, 区别是谁调度

kernel-space的线程也叫LWP, python的Threading库产生的是原生的thread, 也就是kernel-space thread, 因为它是被OS(kernel)所调度的, 是抢占式的, 所以

意味着并行是可以的, 也就是一个线程一个CPU, 但是由于GIL的问题, 所以就算一个线程一个CPU依然不能真正的并发.

user-thread是应用程序自己产生的线程, kernel并不知道这些线程, 所以调度是由应用程序调度的, 至于怎么调度, 可以自己实现抢占式的或者协作式的, 协作式也就是协程了~~~~

The term "light-weight process" variously refers to user threads or to kernel mechanisms for scheduling user threads onto kernel threads.

https://en.wikipedia.org/wiki/Thread_%28computing%29#Processes.2C_kernel_threads.2C_user_threads.2C_and_fibers

https://www.quora.com/How-does-thread-switching-differ-from-process-switching

http://stackoverflow.com/questions/5440128/thread-context-switch-vs-process-context-switch

http://stackoverflow.com/questions/12630214/context-switch-internals


GIL以及GIL扑打(thrashing)效应
================================

由于python的线程都是lwp, 所以是可以并发的，只是没拿到gil的线程是不能运行的啦


thread1 -----running---> release gil　-----wait gil---->

thread2 -----wait gil--> got gil      ------running----->

thread3 -----wait gil--> still wait   ------wait gil---->

**以下都是David Beazley的文章**

1.  http://www.dabeaz.com/python/GIL.pdf
----------------------------------------------

老的GIL，CPU密集型是100个ticks去释放一次GIL，I/O密集型则是执行IO操作的时候释放GIL

1.1  py3.2之前 GIL，两个cpu密集型的线程，两核比单核慢很多。

1.2  有时候ctrl+c不能杀死进程，这是因为如果解释器接收到信号之后，是每一个tick就释放gil让其他线程执行, 直到当前执行的线程为主线程为止. 若主线程被不可中断的thread join或者lock给阻塞了，然后主线程就不会被OS给唤醒，也就是不会重新启动.

     这个时候程序由于check很频繁，运行就很慢!

     1.2.1 The reason Ctrl-C doesn't work with threaded programs is that the main thread is often blocked on an uninterruptible thread-join or lock

     1.2.2 Since it's blocked, it never gets scheduled to run any kind of signal handler for it

     1.2.3 And as an extra little bonus, the interpreter is left in a state where it tries to thread-switch after every tick (so not only can you not interrupt your program, it runs slow as hell!)

1.3  多核情况下慢的原因是释放GIL之后的信号处理上

     1.3.1 GIL thread signaling is the source of that

     1.3.2 After every 100 ticks, the interpreter

           3.3.2.1 Locks a mutex

           3.3.2.2 Signals on a condition variable/semaphore where another thread is always waiting

           3.3.2.3 Because another thread is waiting, extra pthreads processing and system calls get triggered to deliver the signal

1.4  线程切换还依赖于OS的切换，在一般OS中，cpu密集型线程是低优先级，而IO密集型线程是高优先级

1.5  CPU竞态，也就是多核下，两个线程(CPU密集型)同时运行，然后第一个释放掉GIL之后，在信号发给另外一个线程的时候，第一个线程有获取了GIL，这是因为第一个线程还在第一个核上运行，

     而第二个线程就一直获取失败，这样另一个线程就过很久才能拿到GIL(参考之后的convoy effect)

1.6  一个cpu密集的线程和一个IO密集的线程分别在不同核心上运行，然后，跟上面一个情况一样，一旦cpu密集型线程拿到GIL，另外一个线程几乎很难拿到GIL

2.  http://www.dabeaz.com/python/NewGIL.pdf
---------------------------------------------

Py3.2之后GIL被重写了，cpu密集型的线程释放GIL不再是基于tick数目了，但是IO线程切换还是在陷入IO的时候释放GIL。sys.setcheckinterval不再影响线程切换了，而是这个sys.setswitchinterval函数设置解释器切换线程的间隔，默认是0.005s.

由于python的thread是kernel lwp, 那么这个函数应该是调用了sched_setscheduler或者sched_rr_get_interval了.

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

3.  http://www.dabeaz.com/python/UnderstandingGIL.pdf
---------------------------------------------------------

新GIL之后依然存在convoy effect。一个cpu密集型线程和一个io密集型线程同时在多核上运行，这样io密集的线程性能将严重下降，原因是，如果io密集型线程进行io操作的时候，会释放掉GIL，然后cpu密集型的线程拿到

GIL，然后在下一个等待超时后将GIL还给io密集型的线程，但是若io密集型的线程的io操作是不需要挂起的呢，比如write数据情况下，由于os有读写缓冲区(假设空间足够)，所以write不会被阻塞，但是线程还是会释放掉

GIL，然后cpu密集型线程就运行了，这样io密集型的线程必须等待下一个等待超时才能获取GIL，这样性能就下降了。

(一个额外的参考，curio和trio这两个Python异步框架，curio的思想是每一个io操作都会引发yield，而trio中，有些不需要阻塞的io操作，则不会yield, 类比GIL的例子，yield就像
切换线程一样)

新的GIL消除了gil battle, 但是引入了timeout这样一个时间消耗, 所以对于高负载的io应用来说, gil timeout有可能会影响响应时间.

3.1 t2中某个fd可读, 然后t2先等待gil timeout, 然后t1是cpu绑定的,
   
    自然不会释放(假设在5毫秒内), 然后t2强制t1释放gil, 然后t1释放, t2拿到gil, 然后运行, 整个过程中t2第一次等待timeout很有可能是失败的，只能强制让t1让出gil

3.2 接3.1的例子, t3是另外一个线程, 然后os唤醒的不是t2而是t3, 那t2只能再继续竞争

3.3 接3.1的例子, 如果t2的io操作可以立即执行完成, 比如发送缓存区的大小大于发送的数据, 则write可以很快完成, 但是io必须释放gil, 所以t1又拿到了, 如果t2有很多io, 但是
    
    大多数都可以几乎立即执行完成的情况下, 释放gil, 再重新获取gil的timeout就变得很多了 

3.4 A Possible Solution: 

    - If a thread is preempted by a timeout, it is penalized with lowered priority (bad thread)

    - If a thread suspends early, it is rewarded with raised priority (good thread)

    - High priority threads always preempt low priority threads


4.  https://bugs.python.org/issue7946
-------------------------------------------

convoy effect的issue

os中的convoy effect: http://www.geeksforgeeks.org/convoy-effect-operating-systems/


