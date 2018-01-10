网络编程
=============

non-blocking, blocking, timeout, sync, async
----------------------------------------------

python的socket有三种模式, blocking, non-blocking, timeout

socket创建默认是blocking, 进程挂起直到io完成或者异常. 也可以设置socket为non-blocking, 若socket不能立即完成io操作, 则马上返回一个错误.
调用socket.setblocking(0)设置socket为non-blocking.

也可以调用socket.settime(float)设置socket超时时间, timeout是表示在超时时间内, 若io操作不能完成, 则引发timeout错误.

socket模块文档原文:

Some notes on socket blocking and timeouts: A socket object can be in one of three modes: blocking, non-blocking, or timeout. Sockets are always created in blocking mode. In blocking mode, operations block until complete or the system returns an error (such as connection timed out). In non-blocking mode, operations fail (with an error that is unfortunately system-dependent) if they cannot be completed immediately. In timeout mode, operations fail if they cannot be completed within the timeout specified for the socket or if the system returns an error. The setblocking() method is simply a shorthand for certain settimeout() calls.

After fork
-----------
http://stackoverflow.com/questions/31467768/how-to-correctly-use-socket-close-for-fork
.. code-block:: c

    sockfd = socket(...)
    bind(...);
    listen(...);
    
    while(1) {
      new_sockfd = accept(...);
      if(fork() == 0) {
        // Child process
        dosomething(...);
    
      }
      else {
    
      }
    }

When a process forks, file descriptors are duplicated in the child process. However, these file descriptors are distinct from each other. Closing a file descriptor in the child doesn't affect the corresponding file descriptor in the parent, and vice versa.

In your case, since the child process needs the accepted socket new_sockfd and the parent process continues to use the listening socket sockfd, the child should close(sockfd) (in your if block; this doesn't affect the parent) and the parent should close(new_sockfd) (in your else block; this doesn't affect the child). The fact that the parent and child are running at the same time doesn't affect this.

为什么close父进程/子进程中的fd并不会影响到另外一个进程, 这是因为: http://stackoverflow.com/questions/409783/socket-shutdown-vs-socket-close/598759#598759

Calling close and shutdown have two different effects on the underlying socket.

The first thing to point out is that the socket is a resource in the underlying OS and multiple processes can have a handle for the same underlying socket.

When you call close it decrements the handle count by one and if the handle count has reached zero then the socket and associated connection goes through the normal close procedure (effectively sending a FIN / EOF to the peer) and the socket is deallocated.

The thing to pay attention to here is that if the handle count does not reach zero because another process still has a handle to the socket then the connection is not closed and the socket is not deallocated.

On the other hand calling shutdown for reading and writing closes the underlying connection and sends a FIN / EOF to the peer regardless of how many processes have handles to the socket. However, it does not deallocate the socket and you still need to call close afterward.


python socket source_address
-------------------------------

python3.2+才有source_address选项

socket.create_connection中有个source_address的参数

.. code-block:: python

    def create_connection(address, timeout=_GLOBAL_DEFAULT_TIMEOUT,
                          source_address=None):
        host, port = address
        err = None
        for res in getaddrinfo(host, port, 0, SOCK_STREAM):
            af, socktype, proto, canonname, sa = res
            sock = None
            try:
                sock = socket(af, socktype, proto)
                if timeout is not _GLOBAL_DEFAULT_TIMEOUT:
                    sock.settimeout(timeout)
                if source_address:
                    sock.bind(source_address)
                sock.connect(sa)
                # Break explicitly a reference cycle
                err = None
                return sock
    
            except error as _:
                err = _
                if sock is not None:
                    sock.close()
    
        if err is not None:
            raise err
        else:
            raise error("getaddrinfo returns an empty list")

getaddrinfo
~~~~~~~~~~~~~~~

其中getaddrinfo是获取host和port所有可能的socket信息, socket信息由5个参数决定: host, port, family, type, prot:

getaddrinfo的文档 https://docs.python.org/3/library/socket.html#socket.getaddrinfo

Translate the host/port argument into a sequence of 5-tuples that contain

all the necessary arguments for creating a socket connected to that service.

host is a domain name, a string representation of an IPv4/v6 address or

None. port is a string service name such as 'http', a numeric port number or

None. By passing None as the value of host and port, you can pass NULL to

the underlying C API.

The family, type and proto arguments can be optionally specified in order to

narrow the list of addresses returned. Passing zero as a value for each of

these arguments selects the full range of results.

比如:

.. code-block:: python

    In [91]: socket.getaddrinfo('127.0.0.1',80)
    Out[91]: 
    [(<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_STREAM: 1>,
      6,
      '',
      ('127.0.0.1', 80)),
     (<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_DGRAM: 2>,
      17,
      '',
      ('127.0.0.1', 80)),
     (<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_RAW: 3>,
      0,
      '',
      ('127.0.0.1', 80))]

127.0.0.1:80这个地址有三个可能的socket, 分别是ipv4的流数据socket(AF_INET, SOCK_STREAM)等等

一般创建连接的时候host和port是远程地址, 比如:

.. code-block:: python

    In [7]: socket.getaddrinfo('baidu.com', 80, 0, socket.SOCK_STREAM)
    Out[7]: 
    [(<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_STREAM: 1>,
      6,
      '',
      ('123.125.114.144', 80)),
     (<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_STREAM: 1>,
      6,
      '',
      ('220.181.57.217', 80)),
     (<AddressFamily.AF_INET: 2>,
      <SocketKind.SOCK_STREAM: 1>,
      6,
      '',
      ('111.13.101.208', 80))]

所以baidu.com这个host中, SOCK_STREAM的地址有上面这三个, 遍历创建对应的socket类型, 比如创建一个SOCK_STREAM的socket, 然后bind, connect去连接host和port建立起连接

bind
~~~~~~

通过getaddrinfo得到所有的socket信息, 然后创建一个对应类型的socket, 然后bind, connect

调用bind方法绑定到source_address, 其中bind的文档https://docs.python.org/3/library/socket.html#socket.getaddrinfo

Bind the socket to address. The socket must not already be bound. (The format of address depends on the address family — see above.)

然后在https://docs.python.org/3/howto/sockets.html#creating-a-socket 中对于bind有:

**A couple things to notice: we used socket.gethostname() so that the socket would be visible to the outside world.** If we had used s.bind(('localhost', 80)) or s.bind(('127.0.0.1', 80))

we would still have a “server” socket, but one that was only visible within the same machine. s.bind(('', 80)) specifies that the socket is reachable by any address the machine happens to have.

bind就是是否绑定到本机公开地址中, 比如bind('',80)是绑定到0.0.0.0:80, 这样所有的人都可以访问到, 如果bind('127.0.0.1', 80)就是本机地址, 然后你可以绑定到本机地址.

所以source_address就是去连接host, port的源地址, 比如如果bind('0.0.0.0', 5673)就是使用本机的5673这个端口和host, port通信, 比如是自己写的服务器, 自己就可以bind, 然后listen, 等待别人通信. 

如果你不指定source_address的话, 自动分配.

.. code-block:: python

    In [1]: import socket
    
    In [2]: res=socket.getaddrinfo('baidu.com', 80, 0, socket.SOCK_STREAM)[0]
    
    In [3]: af, socktype, proto, canonname, sa = res
    # sa是baidu的地址 
    In [4]: sa
    Out[4]: ('111.13.101.208', 80)
    
    In [5]: sock = socket.socket(af, socktype, proto)
    
    # 这个新创建的socket默认是0.0.0.0:0, 没有绑定本地地址
    In [6]: sock
    Out[6]: <socket.socket fd=11, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=6, laddr=('0.0.0.0', 0)>
    
    # 连接
    In [7]: sock.connect(sa)
    
    # 如果没有指定本地地址, 那么随机给端口, 地址的话是本机地址(有可能是内网地址), 这里给出了10.21.2.245, 是个内部地址, 端口是57772
    In [8]: sock
    Out[8]: <socket.socket fd=11, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=6, laddr=('10.21.2.245', 57772), raddr=('111.13.101.208', 80)>




