list
========


如果一个list只有一个元素, 然后rpop/lpop之后，这个list就会被删除. 

然后如果你之前在list上有设置了过期时间的话，rpop/lpop之后，这个list就会被删除

然后再push的话，此时虽然key同名，但是其实是一个新的list, 所以就会发现某个名字的list的过期时间莫名其妙就没了~~~~~


pipeline
===========


根据redis的文档, 一个连接内如果执行了pipeline, 那么当前连接就不能做其他事情了, 也就是比如多线程下

a线程pipeline了, b线程执行get, b就会阻塞住, 这么也可以理解.


然后不要滥用pipeline.

rdb和aof
============

https://redis.io/topics/persistence

rdb是全盘镜像, aof是修改的日志(mysql备份也是日志)

两者都是cow(copy-on-write), 区别是操作的数据

1. rdb(bgsave)的时候, child会把数据集(dataset)写入到临时文件, 其中就算有修改, 那么主线程则不会

   修改dataset, 而是写在另外一个区域, 然后当child完成操作的时候, 把临时文件替换老的rdb文件

   但是同时主进程依然可以提供写操作

2. aof也是一样, for一个子进程, 然后子进程把修改的指令写入到一个临时的aof文件

   同时如果主进程收到写操作, 那么新的修改指令将会写到一个buffer中

   然后子进程写完临时的aof文件的时候, 通知主进程, 主进程把buffer中的新的修改命令加入到临时aof的最后

   然后主进程把临时aof替换掉老的aof文件

并且, aof和rdb保存两个操作不会同时进行, 否则会有很大的io, 所以redis会把操作进行入队

*Redis >= 2.4 makes sure to avoid triggering an AOF rewrite when an RDB snapshotting operation is already in progress, or allowing a BGSAVE while the AOF rewrite is in progress. This prevents two Redis background processes from doing heavy disk I/O at the same time*

redis重启的时候, 会优先从aof文件中恢复数据, 因为aof是"最完整的"

*In the case both AOF and RDB persistence are enabled and Redis restarts the AOF file will be used to reconstruct the original dataset since it is guaranteed to be the most complete.*

aof优先的话也可以从psync过程中看出来, 看下面

redis主从备份
==================

https://redis.io/topics/replication


master可以不配置持久化
------------------------

master不配置持久化，只在slave配置持久化，但是restart master就完了.

*It is possible to use replication to avoid the cost of having the master write the full dataset to disk: a typical technique involves configuring your master redis.conf to avoid persisting to disk at all, then connect a slave configured to save from time to time, or with AOF enabled. However this setup must be handled with care, since a restarting master will start with an empty dataset: if the slave tries to synchronized with it, the slave will be emptied as well.*

所以一旦关闭master的持久化，就不能配置master开机的时候auto start, 进程管理的restart, 比如systemd, supervisor, 在sentinel里面也一样

有可能master restart太快以至于sentinel没有检测到master失败. 太快也是一种苦恼


full sync/partial sync
--------------------------

redis的主从备份如果不是同aof的话会有问题.

redis在slave restar之后总是会full sync(全量同步)的问题: https://github.com/antirez/redis/issues/4102, 里面有人提到加载aof文件的时候应该从rdb中读取repli_id:offset或者存储repli_id:offset到aof文件

影响v4.0.0, 包括v4.0.1. 也就是psync中，slave只是存储repli_ud:offset到rdb文件，如果slave开启了appendonly的话, slave restar的时候只会读取appendonly, 但是不会从rdb里面

读取repli_id:offset, 所以salve restart的时候,每次都是DB loaded from appendonly file Partial resynchornization not possible (no cached master)

这里就是说加载aof的时候找不到replic_id:offset, 解决嘛，要么在slave中关掉aof, 要么作者在github中建议:

Hello, not sure what loglevel are you using, but I would like to see in the slave "DB loaded from disk..." to verify that the RDB gets loaded. Later actually we see an AOF rewrite as @badboy noted,

so it looks like you are not reading from the RDB. If you want a slave with AOF, but still you want to perform a restart, you should:

1. Terminate the slave with SHUTDOWN SAVE

2. Temporarily disable AOF from the slave configuration.

3. Restart the slave. Wait for partial resync and so forth.

4. Re-enable the AOF with CONFIG SET appendonly yes, and CONFIG REWRITE in redis-cli.

**Currently the PSYNC info cannot be persisted in AOF files (but will be later in next releases).**

master-slave中psync2非一致性问题: https://github.com/antirez/redis/issues/4316

无论slave是否开启aof, master restart之后都会full sync, 原因是每次master restart之后都会生成一个新的replication_id:offset

https://github.com/antirez/redis/issues/4102, 也就是master会从rdb中获取, 数据但是不会去获取replication_id:offset信息~~

redis psync(v1/v2): https://gist.github.com/antirez/ae068f95c0d084891305

psync是加入replication id和offset, psync2是支持当master挂掉，然后集群中的其中一个slave被选举成新的master之后，集群中其他slave配置指向新的master之后，无需full sync了.

PSYNC was not good enough when there was a failover. If a slave is promoted to master, the slaves that replicated with the old master were not able to connect to the new promoted slave and PSYNC with it: a full resynchronization was needed. This is not great, and is not good for Redis Cluster as well.

https://community.centminmod.com/threads/redis-4-0-stable-release.12279 

  1) PSYNC2, the new replication engine. The way the replication handshake and changes propagation happens between masters and slaves was significantly changed. Now slaves promoted to masters are able to 
accept the old slaves (reconfigured to point to the new master) without the need to do a full resynchronization. Similarly slaves can be stopped, upgraded and restarted, and can continue with the master with just a partial resynchronization. People that run Redis operations will appreciate this change... 


expire key in slave
=====================

如果slave也设置expire的话，master已经master和slave之间的时钟同步都会有问题, redis的slave并没有设置expire, 是使用一下三点:

1. slave不会设置key过期时间，只有key在master过期之后, master发送一个del给slave

2. master的del命令会延迟，毕竟master和slave之间的同步是异步的, del命令肯定会有不及时的情况, 所以在slave上的读会有脏读, 所以slave会有自己的logically expired, 一旦一个client

   读取某个key的时候(注意，是读取的时候才判断, only for read operations), 如果是logically expire, slave返回空. 

   However because of master-driven expire, sometimes slaves may still have in memory keys that are already logically expired, since the master was not able to provide the DEL command in time.

   In order to deal with that the slave uses its logical clock in order to report that a key does not exist only for read operations
   
   that don't violate the consistency of the data set (as new commands from the master will arrive). In this way slaves avoid to report logically expired keys are still existing

   如果master上有太多过期的key, master的定期删除忙不过来的情况下, 有些key在slave上依然能访问到: https://toutiao.io/posts/211pvx/preview, 不过这个已经bugfix了

3. lua脚本有关


redis distributed lock
==========================

https://redis.io/topics/distlock

redis实现的分布式锁, 称为Redlock

分布式锁的保证:

* Safety property: Mutual exclusion. At any given moment, only one client can hold a lock.

  互斥性, 一次只能有一个客户端拿到锁

* Liveness property A: Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.

  死锁, 这里没太明白, 大意是总是可以拿到锁, 及时锁住最资源的客户端崩溃或者分区了

* Liveness property B: Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.

  错误容忍, 只要提供锁服务的集群(这里是nodes, 看成集群比较好一点)还可用(比如集群中只要超过半数的nodes是可用的), 那么总是可以获取和释放锁


Why failover-based implementations are not enough
---------------------------------------------------

这一节就是说明了, 一般的锁模式还不够. 比如set key, 然后设置过期时间, 这就是一个锁, 然后删除它才算释放锁

但是, 在master-slave模式下, 当maste丢失的时候, 就会出现问题, 保证不了上面三个特性, 比如

1. Client A acquires the lock in the master.

   客户端A向master请求锁

2. The master crashes before the write to the key is transmitted to the slave.

   master在把这个key(也就是锁)传给(备份模式下)slave之前, master崩溃了

3. The slave gets promoted to master.

   slave被提升为master

4. Client B acquires the lock to the same resource A already holds a lock for. SAFETY VIOLATION!

   然后客户端B依然能获取同一个锁, 因为slave没有收到key, 所以出现A和B都能获取锁的情况

Correct implementation with a single instance
----------------------------------------------------

正确地实现锁, 也就是setnx key value, value是一个随机值, 加上一个过期时间, 不加过期时间的话, set的锁永远不能释放了

1. setnx能保证有锁的话不能设置锁
   
2. 然后随机值是, 有可能客户端A执行卡主了, 超过了锁过期时间, 此时客户端B是可以上锁的, 然后客户单A回来了, 如果直接删除的话
   
   那么会把B的锁值给删掉, 所以需要校验下锁的值和自己的值是否一样, 能删除锁的只能是value同样值的客户端

   so the lock will be removed only if it is still the one that was set by the client trying to remove it

所以, set中带上的过期时间, 则被称为锁的有效时间(lock validity time), 既是锁自动释放的时间, 也就是客户端获取锁之后应该最多执行的时间(毕竟锁会自动释放嘛)


The Redlock algorithm
----------------------

需要对集群中, 超过半数的节点加上锁, 才算获取了锁, 单个节点上锁的过程和上一节一样

In order to acquire the lock, the client performs the following operations:

要获取锁, 当前客户端必须:

1. It gets the current time in milliseconds.

   获取当期那时间毫秒

2. It tries to acquire the lock in all the N instances sequentially, using the same key name and random value in all the instances.

   如果需要对key加锁, 那么使用key和一个随机值, 顺序地向N个实例请求锁(也就是同一组key, random, 依次向实例进行请求)
   
   During step 2, when setting the lock in each instance, the client uses a timeout which is small compared to the total lock auto-release time in order to acquire it.
   
   For example if the auto-release time is 10 seconds, the timeout could be in the ~ 5-50 milliseconds range.
   
   This prevents the client from remaining blocked for a long time trying to talk with a Redis node which is down: if an instance is not available, we should try to talk with the next instance ASAP.

   也就是说, 请求加锁的时候, 请求应该加上一个timeout, timeout时间应该比auto-release短得多, 比如如果auto-release时间是10s, 那么请求的timeout是5-10ms, 这样如果

   提供锁的节点挂掉了, 那么我们可以尽快地请求下一个锁节点

3. The client computes how much time elapsed in order to acquire the lock, by subtracting from the current time the timestamp obtained in step 1.
   
   If and only if the client was able to acquire the lock in the majority of the instances (at least 3), and the total time elapsed to acquire the lock is less than lock validity time
   
   the lock is considered to be acquired.

   客户端要自己计算锁经过的时间, 比如请求完成之后, 在进行其他操作的时候计算锁的时间, 比如获取锁之后, 过了11s(差值是减去请求锁时候的时间戳, 在第一步)才执行完操作
   
   那么此时很显然锁已经过期了(auto-release 10s), 只有小于auto-release时间的才算是一直拿着锁

4. If the lock was acquired, its validity time is considered to be the initial validity time minus the time elapsed, as computed in step 3.

   一旦拿到锁了, 那么其有效时间就是auto-release的时间, 减去经过的时间, 比如在3中, 拿到锁之后, 经过了5s完成了操作, 那么客户端此时锁剩下的时间就是10 - 5 = 5s

5. If the client failed to acquire the lock for some reason (either it was not able to lock N/2+1 instances or the validity time is negative)
   
   it will try to unlock all the instances (even the instances it believed it was not able to lock).

   如果不能拿到锁(不能锁住集群中超过半数的节点), 那么解锁所有的节点

   比如集群中有4个节点, 分别锁1, 2, 3, 4个节点, 其中第三个节点失败, 那么解锁1, 2, 4节点

Is the algorithm asynchronous?
------------------------------------


Releasing the lock
----------------------

直接一次解锁所有上锁的节点就好了, 不管是否成功


对Redlock的一些不同观点
==============================

http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

显然, redis的redlock是依赖时间的, 这并不是一个一致性很好的设计

1. lock expire, 锁的过期时间会导致非一致性, 并且没有一个fencing token的方式去保证一致性

2. 一致性依赖于各种时间, 但是时间是不可信的

3. 两个客户端都拿到锁然后导致非一致性的案例, 服务端失效(某个节点时钟乱序导致锁过期), 客户端失效(客户端发生gc)

4. 各种同步假设让redlock变得并不是一个好选择(相比paxos这类的)

5. 结论，各种客套

其中, 1的例子正式Redlock中避开没有说明的情况:

1. A拿到锁, 然后自己做一些很久的操作, 此时锁过期

2. B此时拿到锁, 因为锁过期了, 然后B去处理其他操作, 比如写入db

3. A此时很久的操作结束, 然后同样, 写入db, 假设和B写入的是同一个key, 那么A此时覆盖B

然后文章中建议加入一个fencing token的机制, 比如

在拿到锁之后, 同时拿到一个ticket, 比如A拿到33, 然后B拿到34, 然后B写入db的时候, 记录下当前key的值的ticket是34

当A写入33的时候, 报错, 因为33比34小

然后针对时间不可信的情况, 文章提出了当某几个节点时间不同步的时候, 会导致锁失效等等问题


redis的回复
-------------

http://antirez.com/news/101

作者针对上一篇文章关于redlock不适用与分布式锁的讨论的讨论

总结来说就是: 分布式锁应该有一个timout, 但是针对timeout应该有一个fencing token的机制去避免由于时间上的不可控导致的一致性

然后分布式锁(系统)不应该对时间(timing)上有任何的假设(假设网络延迟是固定的, 假设时间是同步的等等), 可以看看paxos(看得懂的话)




