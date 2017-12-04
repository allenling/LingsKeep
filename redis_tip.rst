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

redis主从备份
==================

how replication works
-----------------------

https://redis.io/topics/replication


master可以不配置持久化
------------------------

master不配置持久化，只在slave配置持久化，但是restart master就完了.

It is possible to use replication to avoid the cost of having the master write the full dataset to disk: a typical technique involves configuring your master redis.conf to avoid persisting to disk at all, then connect a slave configured to save from time to time, or with AOF enabled. However this setup must be handled with care, since a restarting master will start with an empty dataset: if the slave tries to synchronized with it, the slave will be emptied as well.

所以一旦关闭master的持久化，就不能配置master开机的时候auto start, 进程管理的restart, 比如systemd, supervisor, 在sentinel里面也一样，有可能master restart太快以至于sentinel没有检测到master失败. 太快也是一种苦恼


full sync/partial sync
--------------------------

redis的主从备份如果不是同aof的话会有问题.

redis在slave restar之后总是会full sync(全量同步)的问题: https://github.com/antirez/redis/issues/4102, 里面有人提到加载aof文件的时候应该从rdb中读取repli_id:offset或者存储repli_id:offset到aof文件,影响v4.0.0, 包括v4.0.1
也就是psync中，slave只是存储repli_ud:offset到rdb文件，如果slave开启了appendonly的话, slave restar的时候只会读取appendonly, 但是不会从rdb里面读取repli_id:offset, 所以salve restart的时候,每次都是

DB loaded from appendonly file Partial resynchornization not possible (no cached master) # 这里就是说加载aof的时候找不到replic_id:offset

解决嘛，要么在slave中关掉aof, 要么:
Hello, not sure what loglevel are you using, but I would like to see in the slave "DB loaded from disk..." to verify that the RDB gets loaded. Later actually we see an AOF rewrite as @badboy noted, so it looks like you are not reading from the RDB. If you want a slave with AOF, but still you want to perform a restart, you should:

1. Terminate the slave with SHUTDOWN SAVE
2. Temporarily disable AOF from the slave configuration.
3. Restart the slave. Wait for partial resync and so forth.
4. Re-enable the AOF with CONFIG SET appendonly yes, and CONFIG REWRITE in redis-cli.
**Currently the PSYNC info cannot be persisted in AOF files (but will be later in next releases).**

master-slave中psync2非一致性问题: https://github.com/antirez/redis/issues/4316


无论slave是否开启aof, master restart之后都会full sync, 原因是每次master restart之后都会生成一个新的replication_id:offset, https://github.com/antirez/redis/issues/4102, 也就是master会从rdb中获取
数据但是不会去获取replication_id:offset信息~~

redis psync(v1/v2): https://gist.github.com/antirez/ae068f95c0d084891305, psync是加入replication id和offset, psync2是支持当master挂掉，然后集群中的其中一个slave被选举成新的master之后，集群中其他slave配置指向新的
master之后，无需full sync了.

PSYNC was not good enough when there was a failover. If a slave is promoted to master, the slaves that replicated with the old master were not able to connect to the new promoted slave and PSYNC with it: a full resynchronization was needed. This is not great, and is not good for Redis Cluster as well.

https://community.centminmod.com/threads/redis-4-0-stable-release.12279 
  1) PSYNC2, the new replication engine. The way the replication handshake and changes propagation happens between masters and slaves was significantly changed. Now slaves promoted to masters are able to 
accept the old slaves (reconfigured to point to the new master) without the need to do a full resynchronization. Similarly slaves can be stopped, upgraded and restarted, and can continue with the master with just a partial resynchronization. People that run Redis operations will appreciate this change... 


expire key in slave
---------------------

如果slave也设置expire的话，master已经master和slave之间的时钟同步都会有问题, redis的slave并没有设置expire, 是使用一下三点:

1. slave不会设置key过期时间，只有key在master过期之后, master发送一个del给slave

2. master的del命令会延迟，毕竟master和slave之间的同步是异步的, del命令肯定会有不及时的情况, 所以在slave上的读会有脏读, 所以slave会有自己的logically expired, 一旦一个client
   读取某个key的时候(注意，是读取的时候才判断, only for read operations), 如果是logically expire, slave返回空. 
   However because of master-driven expire, sometimes slaves may still have in memory keys that are already logically expired, since the master was not able to provide the DEL command in time.
   In order to deal with that the slave uses its logical clock in order to report that a key does not exist only for read operations that don't violate the consistency of the data set (as new commands from the master will arrive). In this way slaves avoid to report logically expired keys are still existing
   如果master上有太多过期的key, master的定期删除忙不过来的情况下, 有些key在slave上依然能访问到: https://toutiao.io/posts/211pvx/preview, 不过这个已经bugfix了

3. lua脚本有关


redis distributed lock
==========================


https://redis.io/topics/distlock

http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

redis的redlock不是一个一致性很好的设计

1. lock expire, 锁的过期时间会导致非一致性, 并且没有一个fencing token的方式去保证一致性

2. 一致性依赖于各种时间, 但是时间是不可信的

3. 两个客户端都拿到锁然后导致非一致性的案例, 服务端失效(某个节点时钟乱序导致锁过期), 客户端失效(客户端发生gc)

4. 各种同步假设让redlock变得并不是一个好选择(相比paxos这类的)

5. 结论，各种客套

http://antirez.com/news/101

作者针对上一篇文章关于redlock不适用与分布式锁的讨论的讨论



总结来说就是: 分布式锁应该有一个timout, 但是针对timeout应该有一个fencing token的机制去避免由于时间上的不可控导致的一致性, 然后分布式锁(系统)不应该对
时间(timing)上有任何的假设(假设网络延迟是固定的, 假设时间是同步的等等), 可以看看paxos(看得懂的话)

