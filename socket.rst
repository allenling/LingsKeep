Python Socket Keep
==================

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
