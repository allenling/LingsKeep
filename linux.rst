Threading.Queue和multiprocessing.Queue
==========================================

python原生的Threading.Queue是线程级别的, 用于在一个进程中不同的线程, 可以正常的put和get, 但是不同进程中, 就不行.

multiprocessing的Queue是启动一个新线程去, 在新线程中将put的数据写入到pipe中, 关键词: pipe, 线程

pipe也就是重定向而已, 一个进程的输出转成另外一个进程的输入. 每一个进程中0,1,2就是标准输入, 标准输出, 标准错误, 将标准输出重定向到一个fd,os.dup2(fd, 1), 将标准输入重定向到一个fd, os.dup2(fd, 0)

然后一个进程的输除就能变成另外一个进程的输入了.

daemon进程
===============

https://segmentfault.com/a/1190000008556669

https://www.ibm.com/developerworks/cn/linux/1702_zhangym_demo/index.html

设置守护进程的时候, 有些程序是fork一次, 有些是fork两次

简单来说呢, 就是第一次是为了setid, 因为setid的进程不能是process group leader

fork, 然后杀死父进程, 然后子进程调用setid之后, 就新建了一个会话, 这个会话并没有关联终端, 此时子进程脱离了终端

但是此时(子)进程仍然是session leader, 还是可以主动去关联终端的, 所以再次fork一次, 杀死父进程, 然后这样，(子子)进程就不能打开终端了

第二次fork也可以开终端设备的时候指定O_NOCTTY来避免打开控制终端.


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


