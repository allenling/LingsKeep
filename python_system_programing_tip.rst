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

