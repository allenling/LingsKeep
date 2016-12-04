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


在一些系统调用中需要指定时间是用CLOCK_MONOTONIC还是CLOCK_REALTIME，以前总是搞不太清楚它们之间的差别，现在终于有所理解了。

CLOCK_MONOTONIC是monotonic time

CLOCK_REALTIME是wall time。

monotonic time字面意思是单调时间，实际上它指的是系统启动以后流逝的时间，这是由变量jiffies来记录的。系统每次启动时jiffies初始化为0，每来一个timer interrupt，jiffies加1，也就是说它代表系统启动后流逝的tick数。jiffies一定是单调递增的，因为时间不够逆嘛！
 
wall time字面意思是挂钟时间，实际上就是指的是现实的时间，这是由变量xtime来记录的。系统每次启动时将CMOS上的RTC时间读入xtime，这个值是"自1970-01-01起经历的秒数、本秒中经历的纳秒数"，每来一个timer interrupt，也需要去更新xtime。

以前我一直想不明白，既然每个timer interrupt，jiffies和xtime都要更新，那么不都是单调递增的吗？那它们之间使用时有什么区别呢？昨天看到一篇文章，终于明白了，wall time不一定是单调递增的。因为wall time是指现实中的实际时间，如果系统要与网络中某个节点时间同步、或者由系统管理员觉得这个wall time与现实时间不一致，有可能任意的改变这个wall time。最简单的例子是，我们用户可以去任意修改系统时间，这个被修改的时间应该就是wall time，即xtime，它甚至可以被写入RTC而永久保存。一些应用软件可能就是用到了这个wall time，比如以前用vmware workstation，一启动提示试用期已过，但是只要把系统时间调整一下提前一年，再启动就不会有提示了，这很可能就是因为它启动时用gettimeofday去读wall time，然后判断是否过期，只要将wall time改一下，就可以欺骗过去了。


UNIX Domain Socket
=======================

socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。UNIX域套接字与TCP套接字相比较，在同一台主机的传输速度前者是后者的两倍。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

使用UNIX Domain Socket的过程和网络socket十分相似，也要先调用socket()创建一个socket文件描述符，address family指定为AF_UNIX，type可以选择SOCK_DGRAM或SOCK_STREAM，protocol参数仍然指定为0即可。

UNIX Domain Socket与网络socket编程最明显的不同在于地址格式不同，用结构体sockaddr_un表示，网络编程的socket地址是IP地址加端口号，而UNIX Domain Socket的地址是一个socket类型的文件在文件系统中的路径，这个socket文件由bind()调用创建，如果调用bind()时该文件已存在，则bind()错误返回。


读写默认都是阻塞的，非阻塞需要在send, recv加上一个socket.MSG_DONTWAIT标志.

服务端:

server_address = './uds_socket'

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

sock.bind(server_address)

sock.listen(1)

客户端:

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

try:
    sock.connect(server_address)
except:
    pass



Unix Domain Socket(UDS)和IPC
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

有一个很有用的函数socket.socketpair(), 这里返回可以给父子进程通信的socket， 不需要绑定， 直接使用

master sock1

child sock2

master的写如sock1的时候，数据流向sock2， child只需要从sock2接收数据就行，而child写入sock2， 则数据流向sock1， 这样master也只需要从sock1接收数据就行

master:
    send(sock1, data)
    recv(sock1, size)

child:
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

socketpair会创建两个描述符，但改描述符不属于任何的实际文件系统，而是网络文件系统，虚拟的．同时内核会将这两个描述符彼此设为自己的peer即对端（这里即解决了如何标识读写端，可以想象，两个描述符互为读写缓冲区，即解决了这个问题）．然后应用相应socket家族里的read/write函数执行读写操作．
有了这个基础，即可明白为什么试用fork产生的两个子进程都不关闭读端的时候会竞争，如上所述，他们共享相同的文件表项，有相同的inode和偏移量，两个进程的操作当然是相互影响的．

IPC中，管道(pipe) VS unix domain socket
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

UNIX-domain sockets are generally more flexible than named pipes. Some of their advantages are:

You can use them for more than two processes communicating (eg. a server process with potentially multiple client processes connecting);
They are bidirectional;
They support passing kernel-verified UID / GID credentials between processes;
They support passing file descriptors between processes;
They support packet and sequenced packet modes.
To use many of these features, you need to use the send() / recv() family of system calls rather than write() / read().

Linux Threads vs Light Weight Processes
===========================================

http://www.thegeekstuff.com/2013/11/linux-process-and-threads/

Threads in Linux are nothing but a flow of execution of the process. A process containing multiple execution flows is known as multi-threaded process.

For a non multi-threaded process there is only execution flow that is the main execution flow and hence it is also known as single threaded process. For Linux kernel , there is no concept of thread. Each thread is viewed by kernel as a separate process but these processes are somewhat different from other normal processes. I will explain the difference in following paragraphs.

Threads are often mixed with the term Light Weight Processes or LWPs. The reason dates back to those times when Linux supported threads at user level only. This means that even a multi-threaded application was viewed by kernel as a single process only. This posed big challenges for the library that managed these user level threads because it had to take care of cases that a thread execution did not hinder if any other thread issued a blocking call.

Later on the implementation changed and processes were attached to each thread so that kernel can take care of them. But, as discussed earlier, Linux kernel does not see them as threads, each thread is viewed as a process inside kernel. These processes are known as light weight processes.

The main difference between a light weight process (LWP) and a normal process is that LWPs share same address space and other resources like open files etc. As some resources are shared so these processes are considered to be light weight as compared to other normal processes and hence the name light weight processes.

So, effectively we can say that threads and light weight processes are same. It’s just that thread is a term that is used at user level while light weight process is a term used at kernel level.

From implementation point of view, threads are created using functions exposed by POSIX compliant pthread library in Linux. Internally, the clone() function is used to create a normal as well as alight weight process. This means that to create a normal process fork() is used that further calls clone() with appropriate arguments while to create a thread or LWP, a function from pthread library calls clone() with relevant flags. So, the main difference is generated by using different flags that can be passed to clone() function.

Read more about fork() and clone() on their respective man pages.

How to Create Threads in Linux explains more about threads.
