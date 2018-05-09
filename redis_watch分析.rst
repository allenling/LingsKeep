######################
redis中watch的一些分析
######################


参考: https://redis.io/topics/transactions

watch如何使用, 以及 `redis-py <https://github.com/andymccurdy/redis-py>`_ 这个库是如何实现的

watch后面接普通命令
======================

想一下会不会出现这样的情况:

thread1和thread2分别watch有a, b, 然后thread1进行watch a, thread2进行watch b, 双方互相不影响?

1. thread1, watch a

2. thread2, watch b

3. thread3, incr b

4. thread1, set d 1

5. thread1, do other things, but fail

watch是和连接相关的, 用一个连接内, 如果watch后面接的不是multi, 是一般语句, 那么语句直接执行

**所以多线程用的是一个连接的话, watch之后对一般命令不影响**

  *we are asking Redis to perform the transaction only if none of the WATCHed keys were modified.
  
  (But they might be changed by the same client inside the transaction without aborting it. More on this.)*
  
  --- 来自参考


多线程不同的pipeline, 如何watch
===================================

多线程都进行watch, multi的情况, thread1和thread2是 **同一个进程process1**

1. thread1, watch a

2. process2, incr a

3. thread2, multi, incr b, exec

4. thread2, fail

5. thread1, multi, ..., exec, success

显然, process2更改的a, 那么我们希望是影响thread1, 但是其实是影响了watch后面第一个pipeline, 也就是thread2的pipeline

那么应该怎么做? **在redis-py中, 显然是开新连接的来解决这样watch的冲突的**

使用 *redis-cli monitor* 命令可以看到, thread1和thread2的连接(端口)是不一样的

打印redis.StrictRedis._created_connections将会看到是2

所以, 在redis-py中, thread1和thread2互不影响

