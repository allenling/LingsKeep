linux的信号流程
===================

.. [1] http://kernel.meizu.com/linux-signal.html

.. [2] http://cs-pub.bu.edu/fac/richwest/cs591_w1/notes/wk3_pt2.PDF

.. [3] http://devarea.com/linux-handling-signals-in-a-multithreaded-application/#.WpAhGINuaUk>

.. [4] https://stackoverflow.com/questions/6949025/how-are-asynchronous-signal-handlers-executed-on-linux


参考1是主要参考文章

参考2是liunux/unix的信号概述(ppt)

参考3是简单的sigaction的代码例子

task, 进程, 线程, lwp的概念参考linux_nptl.rst

小结
=========

下面从信号处理流程去看task中的结构信息的作用, 这里不涉及调度, 调度参考linux_task_schedule.rst

  *理解信号异步机制的关键是信号的响应时机，我们对一个进程发送一个信号以后，其实并没有硬中断发生，只是简单把信号挂载到目标进程的信号 pending 队列上去，信号真正得到执行的时机是进程执行完异常/中断返回到用户态的时刻。*
  *让信号看起来是一个异步中断的关键就是，正常的用户进程是会频繁的在用户态和内核态之间切换的（这种切换包括：系统调用、缺页异常、系统中断…），所以信号能很快的能得到执行。但这也带来了一点问题，内核进程是不响应信号的，除非它刻意的去查询。所以通常情况下我们无法通过kill命令去杀死一个内核进程。*
  
  --- 参考1

  *See the functions wants_signal() and completes_signal(). The code picks the first available thread for the signal. An available thread is one that doesn't block the signal and has no other signals in its queue. The code happens to check the main thread first, then it checks the other threads in some order unknown to me. If no thread is available, then the signal is stuck until some thread unblocks the signal or empties its queue.*
  
  --- 参考4

1. 发送信号先通过pid拿到线程组, 然后选出其中一个线程/task(这里经测试, 一般是主线程, 除非主线程退出或者主动block了信号), 把信号加入到信号队列中
   这里的信号队列是线程组的shared_pending, 而不是每一个task结构自己的pending队列

2. 然后唤醒对应的task, 唤醒的时候把task的thread_info.flags设置上TIF_SIGPENDING标志位

3. 唤醒是通过发送中断, 内核会强行中断目标task当前的程序, 然后目标task会进入内核态, 通过判断thread_info.flasg的标志位, 发现有信号处理, 则保存当前程序的栈等信息, 然后当前
   程序的地址切换成信号处理函数, 然后返回用户态. 判断是否有信号要处理是在返回用户态的时候判断的

4. 用户态执行完信号处理函数之后, 又切回内核态, 然后内核把原先保存的程序切换出来, 然后返回用户态继续处理之前的程序.

其中3, 4是涉及到中断的概念和内核态/用户态切换的东西, 没搞懂, 但是通过测试, 得出无论当前任务是什么任务(io等待或者cpu计算), 被发送信号之后,

总会被打断, 然后切换执行信号处理函数, 信号处理函数返回之后又继续执行之前的程序, 所以3, 4可以通过测试推测出来. 

.. code-block:: python

    '''
    
    user_mode                       kernel_mode
    
    routine
                
              --be interrupted->    
                                    save routine and
                                    switch stack to signal_handler
    
              <--return usermode-
    
    signal
    handler
    
              ------trap into---->  
                                    switch to routine
    
              <-return usermode---
    
    routine
    
    '''

**最后, 一定要等待signal handler执行完, 无论之前的程序是什么操作, 比如计算, 比如等待事件发生, 都会被终止执行, signal handler返回之后, 之前的程序才能继续执行!!!**

源码分析参考 [1]_

唤醒的是哪个线程的测试
============================

经过测试, 无论主线程是一直占着cpu还是陷入等待(sleep), signal一般唤醒的都是主线程. 下面是测试源码

启动三个线程, 然后main里面分别做计算和等待操作, 对比两者的情况.

.. code-block:: c

    #include<stdio.h>
    #include<unistd.h>
    #include<pthread.h>
    #include <sys/mman.h>
    #include <stdlib.h>
    #include <sys/prctl.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <sys/ioctl.h>
     
    void *threadfn1(void *p)
    {
    	while(1){
    		printf("thread1\n");
    		sleep(2);
    	}
    	return 0;
    }
     
    void *threadfn2(void *p)
    {
        pthread_t   tid;
        tid = pthread_self();
    	while(1){
    		printf("thread2: %ld\n", (long) tid);
    		sleep(2);
    	}
    	return 0;
    }
     
    void *threadfn3(void *p)
    {
        pthread_t   tid;
        tid = pthread_self();
    	while(1){
    		printf("thread3: %ld\n", (long) tid);
    		sleep(2);
    	}
    	return 0;
    }
     
     
    void handler(int signo, siginfo_t *info, void *extra) 
    {
    	int i;
        pthread_t   tid;
        tid = pthread_self();
    	for(i=0;i<10;i++)
    	{
    		puts("signal");
            printf("in %ld\n", (long) tid);
    		sleep(2);
    	}
    }
     
    void set_sig_handler(void)
    {
            struct sigaction action;
     
     
            action.sa_flags = SA_SIGINFO; 
            action.sa_sigaction = handler;
    
            if (sigaction(SIGRTMIN + 3, &action, NULL) == -1) { 
                perror("sigusr: sigaction");
                _exit(1);
            }
     
    }
     
    int main()
    {
    	pthread_t t1,t2,t3;
        pthread_t   tid;
        tid = pthread_self();
        printf("main thread: %ld\n", (long)tid);
    	set_sig_handler();
    	// pthread_create(&t1,NULL,threadfn1,NULL);
    	pthread_create(&t2,NULL,threadfn2,NULL);
    	pthread_create(&t3,NULL,threadfn3,NULL);
        int count = 0;
        // sleep(3600);
        // 下面的while可以换成sleep
        while (1){
            count += 1;
        }
    	pthread_exit(NULL);
    	return 0;
    }

在main中

1. 无论是while 1计算, 还是sleep, 发送signal(*sudo kill -s 37 pid*)之后总是唤醒的总是主线程

2. 只开启一个子线程, 比如子线程2, 然后子线程2中密集计算(while count += 1), 然后主线程sleep, 依然是唤醒主线程.

所以

1. 也就是对主线程调用wants_signal之后, 总是ture.

2. 无论被选择的task正在进行计算或者等待系统调用返回(sleep/select等等),
   内核(complete_signal->signal_wake_up->signal_wake_up_state)总是直接发送中断, 让task执行signal.

根据参考 [2]_的一些解释:

*CPU checks for interrupts after executing each instruction.*

cpu每一执行一个指令之后, 都会去检查中断

*If interrupt occurred, control unit: Determines vector i, corresponding to interrupt, (省略一些步骤), If necessary, switches to new stack by

Loading ss & esp regs with values found in the task state segment (TSS) of current process, (省略一些步骤), Interrupt handler is then executed!*

简单来说就是拿到signal handler的栈什么的和参数, 然后切换执行.

根据参考 [1]_中的解释, 会保存当前执行函数的栈信息什么的, 切换到用户态执行signal handler, 然后回到内核, 然后再执行之前保存的函数.

**所以, 一旦有信号发生, 并且task定义了自己的handler, 那么内核就将让task执行(强制)signal, 然后再切换到signal之前的程序.**

**强制执行是通过发送中断, 无论目标task是否正在运行还是陷入等待状态, 都会收到中断, 然后检查pending的信号, 然后执行.**

signal handler和main中的程序切换的测试
========================================

1. 主线程read等待端口a数据

2. 主线程注册signal handler, 该handler则会去read另外一个端口b, 等待数据

3. 然后发送信号给pid

4. signal handler被执行, 进入read等待b

5. 此时a有数据, 那么主线程的read会被唤醒吗?也就是进入等待之后, 只跟哪个系统调用被唤醒有关?也就是就算
   signal handler进入等待系统调用的状态, 依然是哪个系统调用有返回, 则唤醒哪个程序?

客户端可以在发送信号之前或者之后connect到a, 有两个情况, recv是否是阻塞, 使用recv或者epoll这种.

1. 阻塞的recv调用

下面是一个阻塞的recv函数

.. code-block:: c

    #define MAXLINE 1024
    
    int read_wait(int port) {
        int server_sockfd;//服务器端套接字  
        int client_sockfd;//客户端套接字  
        int len;  
        struct sockaddr_in my_addr;   //服务器网络地址结构体  
        struct sockaddr_in remote_addr; //客户端网络地址结构体  
        int sin_size;  
        char buf[BUFSIZ];  //数据传送的缓冲区  
        memset(&my_addr,0,sizeof(my_addr)); //数据初始化--清零  
        my_addr.sin_family=AF_INET; //设置为IP通信  
        my_addr.sin_addr.s_addr=INADDR_ANY;//服务器IP地址--允许连接到所有本地地址上  
        my_addr.sin_port=htons(port); //服务器端口号  
          
        /*创建服务器端套接字--IPv4协议，面向连接通信，TCP协议*/  
        if((server_sockfd=socket(PF_INET,SOCK_STREAM,0))<0)  
        {    
            perror("socket");  
            return 1;  
        }  
       
            /*将套接字绑定到服务器的网络地址上*/  
        if (bind(server_sockfd,(struct sockaddr *)&my_addr,sizeof(struct sockaddr))<0)  
        {  
            perror("bind");  
            return 1;  
        }  
          
        /*监听连接请求--监听队列长度为5*/  
        printf("listen in %d\n", port);
        listen(server_sockfd, 1);  
          
        sin_size=sizeof(struct sockaddr_in);  
          
        /*等待客户端连接请求到达*/  
        if((client_sockfd=accept(server_sockfd,(struct sockaddr *)&remote_addr,&sin_size))<0)  
        {  
            perror("accept");  
            printf("error in %d\n", port);
            return 1;  
        }  
        printf("accept client %s:%d\n",inet_ntoa(remote_addr.sin_addr), (int)remote_addr.sin_port);  
        len=send(client_sockfd,"Welcome to my server\n",21,0);//发送欢迎信息  
          
        /*接收客户端的数据并将其发送给客户端--recv返回接收到的字节数，send返回发送的字节数*/  
        while(1){
            int len = recv(client_sockfd,buf,BUFSIZ,0);
            if (len >= 0){
                buf[len]='\0';  
                printf("recv %s\n",buf);  
                if(send(client_sockfd,buf,len,0)<0)  
                {  
                    perror("write");  
                    return 1;  
                }  
            }else{
                perror("recv"); 
                printf("%d recv got 0!!\n", port);
                break;
            }
        }  
        close(client_sockfd);  
        close(server_sockfd); 
        printf("close %d\n", port);
        return 0;  
    }


然后在main和signal handler上指定recv不同的端口


.. code-block:: c

    #include <stdio.h>
    #include <unistd.h>
    #include <pthread.h>
    #include <sys/mman.h>
    #include <stdlib.h>
    #include <sys/prctl.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <sys/ioctl.h>
    #include <stdio.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <string.h>
    #include <netinet/in.h>
    #include <stdlib.h>
    #include <errno.h>
    #include <unistd.h>
    #include <arpa/inet.h>
    
    
    void handler(int signo, siginfo_t *info, void *extra) 
    {
    	int i;
        pthread_t   tid;
        tid = pthread_self();
        printf("handler in %ld \n", (long) tid);
        int port = 10005;
        // 这里进入等待系统调用
        // 监听10005端口
        read_wait(port);
    }
    
    
    void set_sig_handler(void)
    {
            struct sigaction action;
     
     
            action.sa_flags = SA_SIGINFO; 
            action.sa_sigaction = handler;
    
            if (sigaction(SIGRTMIN + 3, &action, NULL) == -1) { 
                perror("sigusr: sigaction");
                _exit(1);
            }
     
    }
    
    int main()
    {
        pthread_t   tid;
        tid = pthread_self();
        printf("main thread: %ld\n", (long)tid);
        set_sig_handler();
        int port = 10004;
        // 监听1004端口
        read_wait(port);
        printf("main return\n");
        return 0;
    }

结果是: 一旦进入了signal handler, 那么就会一直执行signal handler, 然后直到signal handler处理完. 然后再进入到

main, 但是main的recv或者accept(取决于你的客户端是先connect之后再发送37信号还是发送信号之后再connect)都会报错, 然后直接结束main

下面是输出

.. code-block:: python

    '''
    main thread: 140464972683008
    listen in 10004
    accept client 127.0.0.1:63195
    handler in 140464972683008 
    listen in 10005
    accept client 127.0.0.1:40075
    recv a                         # 这是signal handler中收到的数据
    recv: Connection reset by peer
    10005 recv got 0!!close 10005  # signal handler执行完毕
    recv: Interrupted system call  # main函数的accept/recv报错
    10004 recv got 0!!close 10004
    main return
    '''

所以内核强行切换到了signal handler, 并且直到signal handler执行完毕才切换到之前的执行程序.

2. 如果recv是select/epoll这种呢?

测试下来也是一样的, signal handler中退出之后会导致之前的程序发生Interrupted system call异常

下面是epoll的处理函数


.. code-block:: c

    int epoll_fun(int port) {
        struct epoll_event ev, events[MAX_EVENTS];
        int nfds, epollfd;
    
        int server_sockfd;//服务器端套接字  
        int client_sockfd;//客户端套接字  
        int len;  
        struct sockaddr_in my_addr;   //服务器网络地址结构体  
        struct sockaddr_in remote_addr; //客户端网络地址结构体  
        int sin_size;  
        // char buf[BUFSIZ];  //数据传送的缓冲区  
        memset(&my_addr,0,sizeof(my_addr)); //数据初始化--清零  
        my_addr.sin_family=AF_INET; //设置为IP通信  
        my_addr.sin_addr.s_addr=INADDR_ANY;//服务器IP地址--允许连接到所有本地地址上  
        my_addr.sin_port=htons(port); //服务器端口号  
    
    
        // read的buffer
        char read_buf[1024];
          
        /*创建服务器端套接字--IPv4协议，面向连接通信，TCP协议*/  
        if((server_sockfd=socket(PF_INET, SOCK_STREAM,0))<0)  
        {    
            perror("socket create");  
            return 1;  
        }
        printf("socket created\n");
       
            /*将套接字绑定到服务器的网络地址上*/  
        if (bind(server_sockfd,(struct sockaddr *)&my_addr,sizeof(struct sockaddr))<0)  
        {  
            perror("socket bind");  
            return 1;  
        }
        printf("socket binded\n");
          
        /*监听连接请求--监听队列长度为5*/  
        printf("listen in %d\n", port);
        listen(server_sockfd, 1);  
          
        sin_size=sizeof(struct sockaddr_in);  
    
        client_sockfd = accept(server_sockfd,(struct sockaddr *)&remote_addr, &sin_size);
        if (client_sockfd == -1) {
            perror("accept error");
            exit(EXIT_FAILURE);
        }
        printf("%d accepted\n", port);
        setNonblocking(client_sockfd);
    
        epollfd = epoll_create1(0);
        if (epollfd == -1) {
            perror("epoll_create1");
            exit(EXIT_FAILURE);
        }
    
        ev.events = EPOLLIN;
        ev.data.fd = client_sockfd;
        if (epoll_ctl(epollfd, EPOLL_CTL_ADD, client_sockfd, &ev) == -1) {
            perror("epoll_ctl EPOLLIN: client_sockfd");
            exit(EXIT_FAILURE);
        }
        int can_return = 0;
        for (;;) {
            if (can_return == 1) {
                break;
            }
            printf("%d epoll wait-----\n", port);
            nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
            if (nfds == -1) {
                printf("%d epoll_wait error: \n", port);
                perror("epoll_wait");
                exit(EXIT_FAILURE);
            }
            printf("%d epoll wait return!\n", port);
    
            for (int n = 0; n < nfds; ++n) {
                if (events[n].data.fd == client_sockfd) {
                    printf("%d recv: ", port);
                    int real_len = read(events[n].data.fd, read_buf, sizeof(read_buf)-1);
                    if (real_len > 0) {
                        for (int i=0; i<real_len; i ++) {
                            printf("%c", read_buf[i]);
                        }
                        printf("\n");
                    }else{
                        printf("%d recv 0\n", port);
                        perror("epoll recv"); 
                        can_return = 1;
                        break;
                    }
                }
            }
        }
        close(client_sockfd);  
        close(server_sockfd); 
        printf("%d return\n", port);
        return 0;  
    }


然后合并到main中


.. code-block:: c

    #include <sys/epoll.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <sys/mman.h>
    #include <stdlib.h>
    #include <sys/prctl.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <sys/ioctl.h>
    #include <stdio.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <string.h>
    #include <netinet/in.h>
    #include <stdlib.h>
    #include <errno.h>
    #include <unistd.h>
    #include <arpa/inet.h>

    void handler(int signo, siginfo_t *info, void *extra) 
    {
    	int i;
        // 这里进入等待系统调用
        printf("in signal handler\n");
        int port = 10005;
        // 调用epoll
        epoll_fun(port);
    }
     
    void set_sig_handler(void)
    {
            struct sigaction action;
     
     
            action.sa_flags = SA_SIGINFO; 
            action.sa_sigaction = handler;
    
            if (sigaction(SIGRTMIN + 3, &action, NULL) == -1) { 
                perror("sigusr: sigaction");
                _exit(1);
            }
     
    }
    
    
    int
    main(void)
    {
    	set_sig_handler();
        int port = 10004;
        epoll_fun(port);
        return 0;
    }


下面是输出

.. code-block:: python

    '''
    socket created
    socket binded
    listen in 10004
    10004 accepted
    10004 epoll wait-----
    10004 epoll wait return!
    10004 recv: 1004
    10004 epoll wait-----
    in signal handler
    socket created
    socket binded
    listen in 10005
    10005 accepted
    10005 epoll wait-----
    10005 epoll wait return!
    10005 recv: 1005
    10005 epoll wait-----
    10005 epoll wait return!
    10005 recv: 10005 recv 0
    epoll recv: Success
    10005 return                         # 这里, signal handler返回了
    10004 epoll_wait error:              # 然后main里面的epoll报粗了
    epoll_wait: Interrupted system call  # main报错的信息
    '''


所以, 之前的程序只会在signal handler返回之后才能继续, 比如下面的例子

main中一直计算, 然后signal handler一个循环, 我们可以看到:

1. 没有发送信号之前, 有一个cpu是100%使用率

2. 发送信号之后, 则计算代码终止, cpu没有100%使用率, 此时进入signal handler

3. signal handler返回, 计算代码继续, cpu又变成了100%使用率

.. code-block:: c

    #include <stdio.h>
    #include <unistd.h>
    #include <pthread.h>
    #include <sys/mman.h>
    #include <stdlib.h>
    #include <sys/prctl.h>
    #include <sys/types.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <sys/ioctl.h>
    #include <stdio.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <string.h>
    #include <netinet/in.h>
    #include <stdlib.h>
    #include <errno.h>
    #include <unistd.h>
    #include <arpa/inet.h>
     
     
    void handler(int signo, siginfo_t *info, void *extra) 
    {
    	int i;
        pthread_t   tid;
        tid = pthread_self();
    	for(i=0;i<20;i++)
    	{
            printf("signal in %ld\n", (long) tid);
    		sleep(2);
    	}
    }
     
    void set_sig_handler(void)
    {
            struct sigaction action;
     
     
            action.sa_flags = SA_SIGINFO; 
            action.sa_sigaction = handler;
    
            if (sigaction(SIGRTMIN + 3, &action, NULL) == -1) { 
                perror("sigusr: sigaction");
                _exit(1);
            }
     
    }
     
    int main()
    {
        pthread_t   tid;
        tid = pthread_self();
        printf("main thread: %ld\n", (long)tid);
    	set_sig_handler();
        int count = 0;
        while (1){
            count += 1;
        }
        printf("main return\n");
    	return 0;
    }


----


kill发送信号
================


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L2936

.. code-block:: c

    /**
     *  sys_kill - send a signal to a process
     *  @pid: the PID of the process
     *  @sig: signal to be sent
     */
    SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
    {
        struct siginfo info;

        info.si_signo = sig;
        info.si_errno = 0;
        info.si_code = SI_USER;
        info.si_pid = task_tgid_vnr(current);
        info.si_uid = from_kuid_munged(current_user_ns(), current_uid());

        return kill_something_info(sig, &info, pid);
    }

这里传入的pid是pid_t类型, 而这个pid_t的定义是在

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/types.h#L22
    typedef __kernel_pid_t		pid_t;


然后搜索一下, 看到似乎这个__kernel_pid_t是跟平台有关的, 没找到x86_64的, 就看到什么安腾(ia)的, 所以

只能以在posix_types下的定义为准了, 是一个int类型

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/uapi/asm-generic/posix_types.h#L28
    #ifndef __kernel_pid_t
    typedef int		__kernel_pid_t;
    #endif

kill_something_info
======================

https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1399

.. code-block:: c

    /*
     * kill_something_info() interprets pid in interesting ways just like kill(2).
     *
     * POSIX specifies that kill(-1,sig) is unspecified, but what we have
     * is probably wrong.  Should make it like BSD or SYSV.
     */
    
    static int kill_something_info(int sig, struct siginfo *info, pid_t pid)
    {
    	int ret;
    
        // 如果pid大于0, 那么会发送到对应的进程中
    	if (pid > 0) {
    		rcu_read_lock();
    		ret = kill_pid_info(sig, info, find_vpid(pid));
    		rcu_read_unlock();
    		return ret;
    	}
    
    	/* -INT_MIN is undefined.  Exclude this case to avoid a UBSAN warning */
    	if (pid == INT_MIN)
    		return -ESRCH;
    
    	read_lock(&tasklist_lock);
        // (pid <= 0) && (pid != -1), 发送信号给pid进程所在进程组中的每一个线程组
    	if (pid != -1) {
    		ret = __kill_pgrp_info(sig, info,
    				pid ? find_vpid(-pid) : task_pgrp(current));
    	} else {
                // pid = -1, 发送信号给所有进程的进程组，除了pid=1和当前进程自己
    		int retval = 0, count = 0;
    		struct task_struct * p;
    
    		for_each_process(p) {
    			if (task_pid_vnr(p) > 1 &&
    					!same_thread_group(p, current)) {
    				int err = group_send_sig_info(sig, info, p);
    				++count;
    				if (err != -EPERM)
    					retval = err;
    			}
    		}
    		ret = count ? retval : -ESRCH;
    	}
    	read_unlock(&tasklist_lock);
    
    	return ret;
    }

kill_pid_info
==================


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1313

.. code-block:: c

    int kill_pid_info(int sig, struct siginfo *info, struct pid *pid)
    {
    	int error = -ESRCH;
    	struct task_struct *p;
    
    	for (;;) {
    		rcu_read_lock();
    		p = pid_task(pid, PIDTYPE_PID);
                // 这里通过pid获取对应的task结构
    		if (p)
                        // 把信号发送到进程
                        // 也就是把信号发送到线程组
    			error = group_send_sig_info(sig, info, p);
    		rcu_read_unlock();
    		if (likely(!p || error != -ESRCH))
    			return error;
    
    		/*
    		 * The task was unhashed in between, try again.  If it
    		 * is dead, pid_task() will return NULL, if we race with
    		 * de_thread() it will find the new leader.
    		 */
    	}
    }

关于pid_task, 则是返回的是线程组的主线程(进程)

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/kernel/pid.c#L305
    struct task_struct *pid_task(struct pid *pid, enum pid_type type)
    {
    	struct task_struct *result = NULL;
    	if (pid) {
    		struct hlist_node *first;
    		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
    					      lockdep_tasklist_lock_is_held());
    		if (first)
    			result = hlist_entry(first, struct task_struct, pids[(type)].node);
    	}
    	return result;
    }

其中是拿到pid->tasks这个数组中, 对应type的头节点, 然后这个节点中包含的是一个task结构, 然后通过计算这个node在

task结构中的偏移量返回task结构的地址(container_of计算). 所以也就是第一个线程的task结构, 也就是主线程(进程)

https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1279

.. code-block:: c

    int group_send_sig_info(int sig, struct siginfo *info, struct task_struct *p)
    {
    	int ret;
    
    	rcu_read_lock();
    	ret = check_kill_permission(sig, info, p);
    	rcu_read_unlock();
    
    	if (!ret && sig)
                // 最后还是调用do_send_sig_info
                // !!!!!注意, 这里最后一个参数是true!!!
    		ret = do_send_sig_info(sig, info, p, true);
    
    	return ret;
    }

https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L1155

.. code-block:: c

    int do_send_sig_info(int sig, struct siginfo *info, struct task_struct *p,
    			bool group)
    {
    	unsigned long flags;
    	int ret = -ESRCH;
    
    	if (lock_task_sighand(p, &flags)) {
                // !!!!这里, 上面最后一个参数是group, 传参的时候传的是true!!!
    		ret = send_signal(sig, info, p, group);
    		unlock_task_sighand(p, &flags);
    	}
    
    	return ret;
    }


__send_signal
================

上面的do_send_sig_info->send_signal最后会调用到__send_signal


https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L994


.. code-block:: c

    static int __send_signal(int sig, struct siginfo *info, struct task_struct *t,
    			int group, int from_ancestor_ns)
    {
    
    
    	struct sigpending *pending;
    	struct sigqueue *q;
    	int override_rlimit;
    	int ret = 0, result;
    
    	assert_spin_locked(&t->sighand->siglock);
    
    	result = TRACE_SIGNAL_IGNORED;
        // !!!判断是否可以忽略信号
    	if (!prepare_signal(sig, t,
    			from_ancestor_ns || (info == SEND_SIG_FORCED)))
    		goto ret;

        // !!注意这里, 这里如果group是true的话
        // 那么pending是t->signal->shared_pendding, 说明是拿线程组中共享的信号队列
        // 如果group不是true, 那么拿的是task自己的pending
    	pending = group ? &t->signal->shared_pending : &t->pending;

        /*
         * Short-circuit ignored signals and support queuing
         * exactly one non-rt signal, so that we can get more
         * detailed information about the cause of the signal.
         */
        result = TRACE_SIGNAL_ALREADY_PENDING;

        // 这里legacy_queue判断, 如果sig是常规信号, 那么是否已经在队列中了, 如果在了就过
        // 如果sig是实时信号, 则可以重复入队
        // 另外一方面也说明了，如果是实时信号，尽管信号重复，但还是要加入pending队列
        // 实时信号的多个信号都需要能被接收到
        if (legacy_queue(pending, sig))
        	goto ret;
        
        result = TRACE_SIGNAL_DELIVERED;
        /*
         * fast-pathed signals for kernel-internal things like SIGSTOP
         * or SIGKILL.
         */
        // 如果是一些强制信号, 那么直接处理
        // 如果是强制信号(SEND_SIG_FORCED)，不走挂载pending队列的流程，直接快速路径优先处理
        if (info == SEND_SIG_FORCED)
            goto out_set;    
        
        /*
         * Real-time signals must be queued if sent by sigqueue, or
         * some other real-time mechanism.  It is implementation
         * defined whether kill() does so.  We attempt to do so, on
         * the principle of least surprise, but since kill is not
         * allowed to fail with EAGAIN when low on memory we just
         * make sure at least one signal gets delivered and don't
         * pass on the info struct.
         */

        // 符合条件的特殊信号可以突破siganl pending队列的大小限制(rlimit)
        // 否则在队列满的情况下，丢弃信号
        // signal pending队列大小rlimit的值可以通过命令"ulimit -i"查看
        if (sig < SIGRTMIN)
        	override_rlimit = (is_si_special(info) || info->si_code >= 0);
        else
        	override_rlimit = 0;
        
        // 没有ignore的信号，加入到pending队列中
        // pending队列的每一个元素都是sigqueue结构
        q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);

        // 加入pending队列
        if (q) {
        	list_add_tail(&q->list, &pending->list);
        	switch ((unsigned long) info) {
        	case (unsigned long) SEND_SIG_NOINFO:
        		q->info.si_signo = sig;
        		q->info.si_errno = 0;
        		q->info.si_code = SI_USER;
        		q->info.si_pid = task_tgid_nr_ns(current,
        						task_active_pid_ns(t));
        		q->info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
        		break;
        	case (unsigned long) SEND_SIG_PRIV:
        		q->info.si_signo = sig;
        		q->info.si_errno = 0;
        		q->info.si_code = SI_KERNEL;
        		q->info.si_pid = 0;
        		q->info.si_uid = 0;
        		break;
        	default:
        		copy_siginfo(&q->info, info);
        		if (from_ancestor_ns)
        			q->info.si_pid = 0;
        		break;
        	}
        
        	userns_fixup_signal_uid(&q->info, t);
        
        } else if (!is_si_special(info)) {
        	if (sig >= SIGRTMIN && info->si_code != SI_USER) {
        		/*
        		 * Queue overflow, abort.  We may abort if the
        		 * signal was rt and sent by user using something
        		 * other than kill().
        		 */
        		result = TRACE_SIGNAL_OVERFLOW_FAIL;
        		ret = -EAGAIN;
        		goto ret;
        	} else {
        		/*
        		 * This is a silent loss of information.  We still
        		 * send the signal, but the *info bits are lost.
        		 */
        		result = TRACE_SIGNAL_LOSE_INFO;
        	}
        }
    
    

        out_set:
        	signalfd_notify(t, sig);
        	sigaddset(&pending->signal, sig);
                // 选择合适的进程来响应信号，如果需要并唤醒对应的进程
        	complete_signal(sig, t, group);
        ret:
        	trace_signal_generate(sig, info, t, group, result);
        	return ret;
            
    }

complete_signal
==================

这里会选择合适的task去唤醒, 调用wants_signal去检查task是否可以处理信号

.. code-block:: c

    static void complete_signal(int sig, struct task_struct *p, int group)
    {
    	struct signal_struct *signal = p->signal;
    	struct task_struct *t;
    
    	/*
    	 * Now find a thread we can wake up to take the signal off the queue.
    	 *
    	 * If the main thread wants the signal, it gets first crack.
    	 * Probably the least surprising to the average bear.
    	 */
        // 注释上说, 先检查主线程是否可以处理信号
        // 如果可以, 主线程处理
    	if (wants_signal(sig, p))
    		t = p;
    	else if (!group || thread_group_empty(p))
    		/*
    		 * There is just one thread and it does not need to be woken.
    		 * It will dequeue unblocked signals before it runs again.
    		 */
    		return;
    	else {
    		/*
    		 * Otherwise try to find a suitable thread.
    		 */
    		t = signal->curr_target;
                // 否则一个一个去遍历线程, 直到找到一个
                // 线程可以处理信号
    		while (!wants_signal(sig, t)) {
    			t = next_thread(t);
    			if (t == signal->curr_target)
    				/*
    				 * No thread needs to be woken.
    				 * Any eligible threads will see
    				 * the signal in the queue soon.
    				 */
    				return;
    		}
    		signal->curr_target = t;
    	}
    
    	/*
    	 * Found a killable thread.  If the signal will be fatal,
    	 * then start taking the whole group down immediately.
    	 */
        // 注释上说, 如果信号是一些致命的信号
        // 那么遍历所有的task, 每个task的pending队列设置上SIGKILL标志位
        // 然后唤醒task, 也就是杀死task
        if (sig_fatal(p, sig) &&
    	    !(signal->flags & SIGNAL_GROUP_EXIT) &&
    	    !sigismember(&t->real_blocked, sig) &&
    	    (sig == SIGKILL || !p->ptrace)) {
    		/*
    		 * This signal will be fatal to the whole group.
    		 */
    		if (!sig_kernel_coredump(sig)) {
    			/*
    			 * Start a group exit and wake everybody up.
    			 * This way we don't have other threads
    			 * running and doing things after a slower
    			 * thread has the fatal signal pending.
    			 */
    			signal->flags = SIGNAL_GROUP_EXIT;
    			signal->group_exit_code = sig;
    			signal->group_stop_count = 0;
    			t = p;
    			do {
                                // 逐个杀死task
    				task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK);
    				sigaddset(&t->pending.signal, SIGKILL);
    				signal_wake_up(t, 1);
    			} while_each_thread(p, t);
    			return;
    		}
    	}
    
    	/*
    	 * The signal is already in the shared-pending queue.
    	 * Tell the chosen thread to wake up and dequeue it.
    	 */
        // 唤醒task
    	signal_wake_up(t, sig == SIGKILL);
    	return;
    }

next_thread
===============

获取task中线程组中的下一个线程

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L558

.. code-block:: c

    static inline struct task_struct *next_thread(const struct task_struct *p)
    {
    	return list_entry_rcu(p->thread_group.next,
    			      struct task_struct, thread_group);
    }

下一个线程就是thread_group.next了, 所以可以推测线程都是通过thread_group连接起来的

wants_signal
==============

判断线程是否可以处理进程


.. code-block:: c

    /*
     * Test if P wants to take SIG.  After we've checked all threads with this,
     * it's equivalent to finding no threads not blocking SIG.  Any threads not
     * blocking SIG were ruled out because they are not running and already
     * have pending signals.  Such threads will dequeue from the shared queue
     * as soon as they're available, so putting the signal on the shared queue
     * will be equivalent to sending it to one such thread.
     */
    static inline int wants_signal(int sig, struct task_struct *p)
    {
        if (sigismember(&p->blocked, sig))
            return 0;
        if (p->flags & PF_EXITING)
            return 0;
        if (sig == SIGKILL)
            return 1;
        if (task_is_stopped_or_traced(p))
            return 0;
        return task_curr(p) || !signal_pending(p);
    }

1. sigismember作用是: *test wehether signum is a member of set.(&p->blocked, sig)* , 也就是是否线程是否block了信号.
   因为线程可以调用sigprocmask/pthread_sigmask去block指定的信号, 如果结果为真, 表示线程屏蔽了信号.
   可以看参考 [3]_
   
2. PF_EXITING表示进程退出状态

3. SIGKILL这个信号是要传递给所有的线程的(这样才能达到kill的目的), 所以返回1

4. task_is_stopped_or_traced线程是否是终止状态

5. task_curr是判断当前线程是否占用cpu, *task_curr - is this task currently executing on a CPU?*

signal_pending
================

先看看函数调用过程

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L313
    static inline int signal_pending(struct task_struct *p)
    {
    	return unlikely(test_tsk_thread_flag(p,TIF_SIGPENDING));
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1536
    static inline int test_tsk_thread_flag(struct task_struct *tsk, int flag)
    {
        // 这里调用task_thread_info去拿task结构的thread_info
    	return test_ti_thread_flag(task_thread_info(tsk), flag);
    }

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/thread_info.h#L77
    static inline int test_ti_thread_flag(struct thread_info *ti, int flag)
    {
    	return test_bit(flag, (unsigned long *)&ti->flags);
    }

而task_thread_info函数则是一般去拿task结构的thread_info

https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L1456

.. code-block:: c

    #ifdef CONFIG_THREAD_INFO_IN_TASK
    static inline struct thread_info *task_thread_info(struct task_struct *task)
    {
    	return &task->thread_info;
    }
    #elif !defined(__HAVE_THREAD_FUNCTIONS)
    # define task_thread_info(task)	((struct thread_info *)(task)->stack)
    #endif

所以, signal_pending则是去寻找task对应的thread_info是否有设置上了TIF_SIGPENDING标志位


block信号
=============

可以使用sigprocmask/pthread_sigmask去block指定的信号, 前者是线程组, 后者是指定的线程.


signal_wake_up
=================

唤醒task


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched/signal.h#L349
    static inline void signal_wake_up(struct task_struct *t, bool resume)
    {
    	signal_wake_up_state(t, resume ? TASK_WAKEKILL : 0);
    }


    // https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c#L661
    /*
     * Tell a process that it has a new active signal..
     *
     * NOTE! we rely on the previous spin_lock to
     * lock interrupts for us! We can only be called with
     * "siglock" held, and the local interrupt must
     * have been disabled when that got acquired!
     *
     * No need to set need_resched since signal event passing
     * goes through ->blocked
     */
    void signal_wake_up_state(struct task_struct *t, unsigned int state)
    {
        // 这里设置task的thread_info的flag是TIF_SIGPENDING
    	set_tsk_thread_flag(t, TIF_SIGPENDING);
    	/*
    	 * TASK_WAKEKILL also means wake it up in the stopped/traced/killable
    	 * case. We don't check t->state here because there is a race with it
    	 * executing another processor and just now entering stopped state.
    	 * By using wake_up_state, we ensure the process will wake up and
    	 * handle its death signal.
    	 */
        // wake_up_state则是去唤醒task!!!!
    	if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
    		kick_process(t);
    }


do_signal/handle_signal
==========================

在内核去唤醒对应的task的时候, task会收到中断, 然后内核判断是信号的话, 则再返回用户态的时候, 把执行的栈什么的信息切换成signal handler, 同时保存当前执行的程序.

切换到用户态的时候会直接执行signal handler.

当收到中断, 返回用户态之前, 调用exit_to_usermode_loop->do_signal->handle_signal

.. code-block:: c

    static void exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags)
    {
    	/*
    	 * In order to return to user mode, we need to have IRQs off with
    	 * none of EXIT_TO_USERMODE_LOOP_FLAGS set.  Several of these flags
    	 * can be set at any time on preemptable kernels if we have IRQs on,
    	 * so we need to loop.  Disabling preemption wouldn't help: doing the
    	 * work to clear some of the flags can sleep.
    	 */
    	while (true) {
    		/* We have work to do. */
    		local_irq_enable();
    
    		if (cached_flags & _TIF_NEED_RESCHED)
    			schedule();
    
    		if (cached_flags & _TIF_UPROBE)
    			uprobe_notify_resume(regs);
    
    		/* deal with pending signal delivery */
                // 去查看是否有信号
    		if (cached_flags & _TIF_SIGPENDING)
    			do_signal(regs);
                    // 省略代码
                    }
            // 省略代码
    }

注意看到_TIF_SIGPENDING这个标志位和TIF_SIGPENDING:

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/arch/x86/include/asm/thread_info.h#L80
    #define TIF_SIGPENDING		2	/* signal pending */
    
    // https://elixir.bootlin.com/linux/v4.15/source/arch/x86/include/asm/thread_info.h#L106
    #define _TIF_SIGPENDING		(1 << TIF_SIGPENDING)

而之前signal_wake_up_state函数中调用的 *set_tsk_thread_flag(t, TIF_SIGPENDING);*, 则是最后调用到set_bit, 看起来是把t这个task

的thread_info中的flags中的第i位置1, 也就是flag中第TIF_SIGPENDING位为1, 也就是100, 也就是等于_TIF_SIGPENDING = 1 << 2.

**上面的过程是推测, set_bit是使用cpu指令的, 没太看懂.**


sigaction
============

  *The original Linux system call was named sigaction().  However, with the addition of real-time signals in Linux 2.2, the fixed-size, 32-bit sigset_t type supported by that system call was no longer
  fit  for  purpose.  Consequently, a new system call, rt_sigaction(), was added to support an enlarged sigset_t type.  The new system call takes a fourth argument, size_t sigsetsize, which specifies
  the size in bytes of the signal sets in act.sa_mask and oldact.sa_mask.  This argument is currently required to have the value sizeof(sigset_t) (or the error EINVAL results).  The glibc sigaction()
  wrapper function hides these details from us, transparently calling rt_sigaction() when the kernel provides it.*
  
  --- sigaction的man手册

根据man手册上的说明, rt_sigaction这个系统调用是取代旧的sigaction系统调用, 并且glibc中的sigaction函数将会调用rt_sigaction这个系统调用

所以, 我们调用sigaction的时候, 其实是调用glibc的sigaction, glibc对一些系统调用进行了wrap, 比如fork和clone.


linux的x86_64架构下的sigaction

sysdeps/unix/sysv/linux/x86_64/sigaction.c


.. code-block:: c

    int
    __libc_sigaction (int sig, const struct sigaction *act, struct sigaction *oact)
    {
      int result;
      struct kernel_sigaction kact, koact;
    
      if (act)
        {
          kact.k_sa_handler = act->sa_handler;
          memcpy (&kact.sa_mask, &act->sa_mask, sizeof (sigset_t));
          kact.sa_flags = act->sa_flags | SA_RESTORER;
    
          kact.sa_restorer = &restore_rt;
        }
    
      /* XXX The size argument hopefully will have to be changed to the
         real size of the user-level sigset_t.  */
      // 这里!!!调用了系统调用rt_sigaction
      result = INLINE_SYSCALL (rt_sigaction, 4,
    			   sig, act ? &kact : NULL,
    			   oact ? &koact : NULL, _NSIG / 8);
      if (oact && result >= 0)
        {
          oact->sa_handler = koact.k_sa_handler;
          memcpy (&oact->sa_mask, &koact.sa_mask, sizeof (sigset_t));
          oact->sa_flags = koact.sa_flags;
          oact->sa_restorer = koact.sa_restorer;
        }
      return result;
    }


glibc的sigaction函数只是帮我们组装了sigaction结构, 然后调用rt_sigaction系统调用.

而rt_sigaction的系统调用是在https://elixir.bootlin.com/linux/v4.15/source/kernel/signal.c找到, 其中会根据宏定义的不同去有不同的实现.

但是本质上, 最终调用的还是do_sigaction这个函数


do_sigaction
================

这个函数的作用是把current, 也就是当前task, 的信号处理函数替换成用户指定的函数


.. code-block:: c

    int do_sigaction(int sig, struct k_sigaction *act, struct k_sigaction *oact)
    {
    	struct task_struct *p = current, *t;
    	struct k_sigaction *k;
    	sigset_t mask;
    
    	if (!valid_signal(sig) || sig < 1 || (act && sig_kernel_only(sig)))
    		return -EINVAL;
    
        // !!!拿到当前task的信号处理函数!!!!!
    	k = &p->sighand->action[sig-1];
    
    	spin_lock_irq(&p->sighand->siglock);
    	if (oact)
    		*oact = *k;
    
    	sigaction_compat_abi(act, oact);
    
    	if (act) {
    		sigdelsetmask(&act->sa.sa_mask,
    			      sigmask(SIGKILL) | sigmask(SIGSTOP));
                // !!!这里替换掉用户指定的信号函数
    		*k = *act;
    		/*
    		 * POSIX 3.3.1.3:
    		 *  "Setting a signal action to SIG_IGN for a signal that is
    		 *   pending shall cause the pending signal to be discarded,
    		 *   whether or not it is blocked."
    		 *
    		 *  "Setting a signal action to SIG_DFL for a signal that is
    		 *   pending and whose default action is to ignore the signal
    		 *   (for example, SIGCHLD), shall cause the pending signal to
    		 *   be discarded, whether or not it is blocked"
    		 */
                // 下面这个判断是该信号是否被ignore
                // sig_handler这个拿到sig的handler, 如果handler是SIG_IGN
                // 那么表示忽略
                // 忽略的时候把所有线程的中的该signale从pending移除
    		if (sig_handler_ignored(sig_handler(p, sig), sig)) {
    			sigemptyset(&mask);
    			sigaddset(&mask, sig);
    			flush_sigqueue_mask(&mask, &p->signal->shared_pending);
    			for_each_thread(p, t)
    				flush_sigqueue_mask(&mask, &t->pending);
    		}
    	}
    
    	spin_unlock_irq(&p->sighand->siglock);
    	return 0;
    }


flush_sigqueue_mask的注释是: Remove signals in mask from the pending set and queue.







