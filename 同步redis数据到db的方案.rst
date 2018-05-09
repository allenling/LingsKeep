redis的数据同步到db的一些情况
==================================


参考python引用计数
=====================

从点赞同步例子看一下

读写分离, 存储的是增量
-------------------------

比如a文章的id是1, 那么a点赞数据的key1=content:like:1, 读取的时候读取这个key1, 然后点赞量增加的时候, incr另外一个key, 称

为key2 = content:like:1:incr, 然后定时周期p, 把key2的数据同步到db, 同步的时候是获取like数据有db_like_a, 然后db_like_a += key2, 然后同时celery去删除key1, 这样的

话, 简单很多了!!!!

.. code-block:: python

    '''

    incr_key=0, key=20 <---- 初始化状态

    incr_key=1, key=20 <---- 点赞加1

    incr_key=1, key=20 <---- celery更新db, db += incr_key = 21

    incr_key=0, key=0  <---- celery删除key

    incr_key=0, key=21 <---- 客户端从db中拉取出个数

    
    '''


有个问题是: key1＝20, key2=0, 然后我点了个赞, key2=1, 然后由于此时没有把key2同步到db, 可能是没有周期时间或者在在同步中但是没轮到key2, 

所以此时, 如果我刷新了一下, 发现key1还是20!!!!!!!也就是我增加了一个赞, 但是居然数据还是原来的~~~

.. code-block:: python

    '''

    incr_key=0, key=20 <---- 初始化状态, 客户端读取到的是20

    incr_key=1, key=20 <---- 点赞加1

    incr_key=1, key=20 <---- 客户端刷新, 读取的时候还是20

    celery再这里才开始更新

    
    '''

**当然, 这种情况是可以一定程度是可以容忍的~~~但是, 有没有其他方案?**

读写都在一个key上
----------------------

读写都在一个key的话, 流程就是

.. code-block:: python

   '''

    key=20 <---- 初始化状态, 客户端读取到的是20

    key=21 <---- 点赞加1

    key=21 <---- celery拿到key, 保存到db

    key=0  <---- 删除key?????????????

    key=21 <---- 客户端拉取出key

    '''
    
    
**关键点在于怎么删除:**

1. 如果加上过期时间的话, 比如在celery拿key的最新值的时候, key过期了, 那么拿到的key就变成了0, 自然db就丢数据了, 甚至会把db的数据污染成0. 如果不加上过期时间, 那么谁删除?

2. 如果是celery删除的话, 怎么删除呢? 在celery同步到db完成的时候, 此时key变成了23, 那么celery删除key的时候, 丢了数据.

3. 应用层自然不能删, 应用层只是get or load

无论1还是2都会找出数据一致性的问题, 并且方式1并不好, 因为我们并不能控制删除的流程, 所以只有选2的方式. 2的方式就是一种主动删除的方案

2的矛盾点在于, **当celery要删除数据的时候, 数据有变更, 此时我们是不能删除的**

所以解决点就是在于, 如果在操作的时候遇到key有变更就放弃, 刚好, redis有watch这样一个命令可以满足我们的需求

如果watch的key有变更, 那么就不能删除

**那么到底什么时候删除?** 可以把需要删除的key放入一个列表, 然后一个定时任务去删除列表中的key.

检测到key不能删除
--------------------

剩下的就是如何判断一个key是否能删除.

如果是watch要删除的key本身, 那么有个问题:

如果celery把key删除之后, 应用层才操作key, 比如celery已经watch了key, 然后把key的值, 比如key=21, 更新到db, 然后删除key, 此时key=0

应用层把key给加1, incr key, 那么此时数据就错了, 因为如果我们再次读取该key的话, 就只能读取到0, 而实际上db中的key的值是21

所以, 如果应用层 **将要** 操作key的话, 那么celery不能删除. 将要表示应用层等下要操作, 那么如何表示呢?

可以参考 **python的引用计数**, 比如对一个key操作之前, 把key的操作计数的key, 比如key_handler, 加1, 操作完成之后, key的操作key, key_handler, 计数减少1

celery不再是watch要操作的key, 而是watch操作的key, 也就是key_handler.

注意的是, 应用层操作key **之前** 要增加key_handler的计数, 操作完成key **之后** 删除key_handler的计数

因为redis是单线程, 所以是有先后顺序, 不考虑同时操作的情况(并发), 一般流程:

.. code-block:: python

    '''
    key=20, key_handler=0  <---- 初始状态
    
    key=20, key_handler=0  <---- 此时celery进行watch的话, watch的key_handler=0
    
    key=20, key_handler=1  <---- 此时客户端增加handler的计数
    
    key=20, key_handler=1  <---- 然后celery保存
    
    key=20, key_handler=1  <---- celery删除key, 此时watch的发现key_handler=1, 也就是key_handler变化了, 删除失败
    
    key=21, key_handler=1  <---- 客户端incr(key), 等待下一次celery的操作
    
    '''


1. 一旦有修改过数据，删除应该被抛弃，可以用一个key_handler的变量表示key有没有客户端在处理, 客户端处理key之前必须incr一下handler_count, 更新完key之后incrby(handler_count, -1)

   handler_count为0表示没有客户端需要更新key或者客户端已经更新完key了，这个时候celery执行删除是安全的.

2. 然后celery就watch这个变量, 这个变量有变化的话，那删除必定是失败的. 那如果再watch之前就被改过，然后直到celery执行delete完之后，客户端都没有机会去incrby(handler_count, -1), 

   或者直接watch报错之后放弃删除


可能的逻辑漏洞
-------------------

.. code-block:: python

    '''
    key=20, key_handler=0  <---- 初始状态
    
    key=20, key_handler=1  <---- 客户端增加handler的计数
    
    key=20, key_handler=1  <---- 此时celery进行watch的话, watch的key_handler=1
    
    key=20, key_handler=1  <---- 然后celery保存
    
    key=0,  key_handler=1  <---- celery删除key, 此时由于redis是单线程, 有可能celery先进行删除操作, 发现key_handler没有变化, 则可以删除
    
    key=1,  key_handler=1  <---- 此时轮到客户端去incr(key), 此时数据丢失
    
    '''

可以看到, 可能的是celery在watch的时候, handler已经变成了1了, 如果之后celery优先删除(redis单线程, 谁先不知道), 那么客户端有可能incr错误的数据.

**所以, 不能删除的还有一个情况就是, key_handler不能大于0!!!**

如果key_handler个数大于0, 那么就是说明有客户端 **想要** 更新key, 那么celery就不去保存和删除了

.. code-block:: python

    '''
    key=20, key_handler=0  <---- 初始状态
    
    key=20, key_handler=1  <---- 客户端增加handler的计数
    
    key=20, key_handler=1  <---- 此时celery进行watch的话, watch的key_handler=1

    key=20, key_handler=1  <---- 此时celery发现key_handler大于0, 则同时放弃保存和删除, 或者可以保存, 但是不能删除?
    
    key=21, key_handler=1  <---- 此时轮到客户端去incr(key), 此时数据丢失
    
    '''

当celery发现key_handler大于0的时候, 可以保存退出? 还是说什么都不做? 都可以, 因为我们没有删除key, 下一次的同步会把最新值给同步到db的.

什么都不做比较好的一点是代码直接return就好了

最终方案
------------

参考了python中的引用计数, 思路是一个对象的引用计数为0, 说明没有人引用它了, 可以安全地回收内存.

在这里表示, 如果一个key的操作计数, 比如key_handler, 的值为0, 表示这个key可以安全地删除.

1. 客户端更新key之前, 需要增加key_handler个数, incr(key_handler)

2. celery获取key之前先watch(key_handler), 获取key的时候同时获取key_handler(pipeline), 如果发现key_handler大于0, 退出

3. 如果celery发现key_handler等于0, 进行save然后delete操作

4. 此时如果客户端完成操作, 减少了key_handler格式, incrby(key_handler, -1), 那么会导致3失败

存在的问题
---------------

上面的方案存在的问题就是

如果客户端在增加和减少key_handler之间如果程序崩溃了, 也就是会出现某个key的key_handler的计数是一直大于0

但是其实已经没有人去更新它了, 这样的key会一直删除不掉

这个可以通过开启一个定时清理任务去处理这些key

同时, 每次增加key_handler的时候, 加一个过期时间, 这个过期时间尽量大, 比如一天什么的

这个过期时间的思路和redis中的锁(setnx以及Redlock)思路差不多, 必须设置一个过期时间, 否则这个key就不能被清理了


多个task进行watch怎么办?
-----------------------------

没关系, 因为连接(connection)不一样, 互不影响的


分散过期任务的搜索
====================

比如业务中, 某一类资源会有过期时间, 然后需要把过期的资源更新到db

1. 过期使用redis的keyspace notification去接收通知, 然后celery去更新过期资源的过期状态, 但是这个方案有点麻烦就是

   a. 要么去写redis的lua脚本, key被删除的时候发送到celery任务
      
   b. 要么写一个后台任务, 挂起然后去pub/sub, 两者都很麻烦
   
2. 定时任务去看是否有过期的任务, 定时的长度为一分钟, 这个方法比起1更简单一点, 定时任务记得去把过期的资源从set(或者sorted set)结构中删除

   这个方法把所有的资源都放到一个sorted set中, 用过期时间去作为得分, 然后celery中使用pipeline包住ZRANGEBYSCORE和

   ZREMRANGEBYSCORE去删除和返回过期的想法. celery定时删除key和更新set/sorted set中的key, 以及资源在数据库中的状态

   问题是, 量大怎么办, 能分散掉吗?


我们可以存储的时候用sorted set, 根据过期时间的日期作为sorted set的key, 比如有5个资源, a, b, c, d, e, 每个过期时间是1, 2, 3, 4, 5天

这里关心日期, 具体到时分秒忽略掉

那么根据当前时间t, 创建5个过期sorted set, t+1, t+2, t+3, t+4, t+5, 当然, 还有一个t的sorted set, 然后

定时任务只搜索当前日期的过期sorted set, 比如今天只搜索t这个过期sorted set, 然后明天只搜索t+1这个sorted set

然后当天的任务同时去校验昨天的过期sorted set, 比如t+1的时候, 校验t这个sorted set, 如果存在, 报警告


多任务导致覆盖
===================


同步redis数据到db的方案
-----------------------------

一般性的, 在celery定时任务中, 设置m时间内去保存数据到db, 这个task是t, 那么

任务t的流程一般是

.. code-block:: python

    data = r.mget(*keys)
    
    for key, value in zip(keys, data):
    
        update(key, value)

其中涉及到:

1. query数据, 比如mget

2. db的update的事务

造成问题主要是: 资源争用和锁设计

分析事务和死锁
-----------------

如果一次性提交事务, 那么

worker1执行任务t没有执行完, 此时有celery beat又会发送任务t, 此时worker2会去处理t, 更新db的时候, worker1和worker2可能会出现死锁, 数据覆盖等问题.

比如任务t是:

.. code-block:: python

    data = r.mget(*keys)
    
    with transaction.atomic():

        for key, value in zip(keys, data):
            update(key, value)

如果:

1. worker1, for key=1, value=1

2. worker2, for key=2, value=2

3. worker1, for key=2, value=2

4. worker2, for key=1, value=1

那么3, 4中就会死锁

可以把事务锁用在for循环里面, 也就是每一个update都是一个事务, 不要一起提交了

.. code-block:: python

    data = r.mget(*keys)
    

    for key, value in zip(keys, data):

        with transaction.atomic():
            update(key, value)

那么有可能出现数据覆盖的问题, 比如

1. worker1, for key=2, value=1

2. worker2, for key=2, value=2

3. worker2, for key=2, value=2

4. worker1, for key=2, value=1

简单方案
------------

简单的话就是整个任务范围呢加锁, 一个时间只能允许一个任务允许, 比如在redis使用setnx加上时间戳, 如果setnx失败, 表示

其他worker在执行t, 那么退出

.. code-block:: python

    task_lock = t + '_lock'

    timestamp = now()

    if setnx(task_lock, timestamp) is False:
        return

    data = r.mget(*keys)
    

    with transaction.atomic():

        for key, value in zip(keys, data):
            update(key, value)

    # 最后记得删除锁

    r.delete(task_lock)

这样的话时间m内, 只能有一个t被执行, 不能利用到多worker的优势了

继续
---------

一般只要限制只有一个worker在运行同一个任务t, 基本上可以了

**如果要利用其多worker呢? 显然, 为了解决资源争用必须加锁, task基本的锁会限制单worker, 所以必须考虑更小粒度的锁**

* 在update到db的时候, 为了减少sql条数, 一般是在for中组建sql语句, 然后使用事务一次性提交.

  这样就会由于两条update语句, 由于加锁顺序不同, 导致死锁, 比如worker1先update 1, 然后update 2, worker2先update 2, 后update 1 

* 那么如果不是一次性提交事务, 而是单条update语句做一个事务, 那么会出现数据覆盖的问题, 比如worker1后update 2, 覆盖了worker2中的update 2

当然, 就算一次性事务提交, 也会出现数据覆盖的问题. **根本原因就是, worker拿到的数据到底是不是最新的? 是否能够去update?**


方案1
-----------

数据覆盖是旧数据覆盖新数据的问题, 那么如果确保自己更新的是最新的数据呢? 可以使用时序来判断

如果操作key的当前最新时间是tim, 自己的操作时间是tim1, 如果tim1 < tim的话, 说明有其他更新的worker在操作, 则放弃update

1. worker1, query key=1, value=1, tim1=1

2. worker1, set 1_latest = tim1, and update key=1

3. worker2, query key=1, value=2, tim2=2

4. worker2. set 1_latest = tim2, and update key=2

执行update的时候, worker1判断1_latest = 2 > tim1 = 1, 那么放弃执行

**注意的是, 这个时序不是worker执行update的时间, 而是从redis拿到数据的时间!**

如果是worker执行update的时间, 那么worker1执行update key=1的时间晚于worker2的话, worker1的tim还是大于worker2, 还是会覆盖的

一个很重要的前提就是: **worker1和worker2谁先执行都有可能**

所以, 引用入一个辅助的key, 每次query之后, 设置该辅助key的值为最新的时间戳, 每次worker执行update之前

判断时间戳是不是最新的, 所以, 如果去设置辅助key的时间戳? 我们希望是query到数据之后, 同时直接设置, 希望能在一个pipeline中执行

.. code-block:: python

   timestamp = now()

   second_keys = [get_second_key(k) for k in keys)

   with r.pipeline() as p:

       values = p.mget(*keys)

       p.mset(*second_keys, timestamp)


但是这样是不行的, 因为必须等mget返回之后, 拿到的key, 再mset, 所以必须是   

.. code-block:: python

   timestamp = now()

   with r.pipeline() as p:

       p.mget(*keys)

       values = p.execute()

   // 辅助key
   second_keys = [get_second_key(k) for k in keys)

   r.mset(*second_keys, timestamp)

但是这样就出现一个问题:

如果在mget和mset之间出现了覆盖呢? 比如

1. worker1, p.mget

2. worker2, p.mget

3. worker2, r.mset

4. worker1, r.mset

所以, 最新的时间戳就被worker1自己的比较小的时间戳给覆盖了, 那么在update的时候, worker2判断时间戳不是自己的, 那么update的流程就是

.. code-block:: python

    for k, v in zip(keys, values):
        latest_t = r.get(get_sconed_key(k))
        if latest_t != timestamp:
            # 这里怎么办???????????/



当然, 我们在判断latest_t !+ timestamp的时候, 再判断如果latest_t < timestamp, 表示自己的时间戳比较大, 那么可以update.

但是,

1. 这样worker1和worker2都能update, 再不知道谁先执行的情况下, 依然不能解决数据覆盖的问题.

2. 还有个问题, 如果判断latest_t < timestamp也可以继续的话, 那么worker2, worker3时序都比worker1大, 那么worker2和worker3也会出现覆盖问题!!!!!!!!!!!

3. 就算使用setnx去更新辅助key的最新时间, 依然不能保证不会数据覆盖.


方案2.0
--------------

如果允许多个worker执行update, 并且在不确定哪个worker先执行的情况下, 总是出现数据覆盖的问题.

**所以, 问题就是, 我们还是无法保证新值不被旧值覆盖, 所以, 可以把思路换成总是更新最新值**

可以这样, 使用一个sorted set记录最新值, sorted set中, sorted set中的key=value, score也是value, 然后在query之后

把对应key的value加入到指定的sorted se中, 每次更新的时候拿出最新的value, 如果和自己的value不一致, 则放弃update


.. code-block:: python

    values = r.mget(*keys)
    
    with r.pipeline() as p:
        
        for key, value in zip(keys, query):

            second_key = get_second_key(key)

            // 加入到辅助sorted set中
            // 注意的是, 得分也是value
            p.zadd(skey, value, value)

比如, key=1, second_key=1_latest, 然后worker1执行mget的是拿到的value=1, 然后执行zadd(1_latest, 1, 1), 同样的

worker2执行mget拿到的key=1, value=2, 然后执行zadd(1_latest, 2, 2)

然后在update之前, 使用zrangebyscore, 获取最新的key, 如果最新的value是不是小于自己拿到的

.. code-block:: python

    for skey, value in zip(keys, values):

       second_key = get_second_key(key)

       max_v = r.zrangebyscore(second_key, '+inf', '-inf', 0, 1)

       if max_v > value:

           continue

       update(skey, value)


**会不会出现worker2被worker1覆盖呢?**

可以用反证法:

如果出现worker1晚于worker2执行, 那么必然worker1调用zrangebyscore拿到的max_v等于自己的value,

那么必然此时worker2没有zadd, 那么worker2比如需要经过zadd才能zrangebyscore, 所以worker2必然是晚于worker1, 矛盾!!

.. code-block:: python

    def update_latest(keys, r):
        '''
        保证最新值被更新到db
        '''
        values = r.mget(*keys)
        second_keys = [k + '_latest' for k in keys]
        with r.pipeline() as p:
            for sk, v in zip(second_keys, values):
                p.zadd(sk, v, v)
                p.expire(sk, 3600 * 24)
        for sk, k, v in zip(second_keys, keys, values):
            max_v = r.zrangebyscore(sk, '+inf', '-inf', 0, 1)
            if max_v != v:
                continue
            # TODO: 更新操作
        return


最后
----------

剩下的问题就是:

1. 辅助的key谁删除?

2. timestamp的一致

关于1, zadd的同时使用pipeline把辅助的key设置一个足够大的过期时间, 比如一天!

.. code-block:: python

    values = r.mget(*keys)
    
    with r.pipeline() as p:
        
        for key, value in zip(keys, query):

            second_key = get_second_key(key)

            // 加入到辅助sorted set中
            // 注意的是, 得分也是value
            p.zadd(skey, value, value)
            p.expire(second_key, 3600 * 24)


关于2, 感觉不是什么问题, 如果真要做强一致性, 也就是所有的时间戳都去一个时间服务去拿了, 那么又是一个大项目了, 先没必要


sorted set和string的区别
-----------------------------

sorted set防止覆盖, 并且能拿到最新的值, string则不能防止覆盖.

获取最新值的时候, 如果使用的是string, 比如key=1, sconed_key=1_latest, worker1拿到的是value1, worker2拿到的是value2, 并且value1 < value2

如果是:

.. code-block:: python
   
    '''
    
    set 1_latest value1(time1)
    
    get 1_latest
    
    '''

的话, 那么就是是更新之前get一下, 还是会出现worker2的值会被worker1的给覆盖, 也就是worker2执行get的时候, 拿到的不是自己的值value2, 而worker1拿value1

那么worker1会执行更新.

1. worker2, set 1_latest value2

2. worker1, set 1_latest value1

3. worker1, get 1_latest, and update

4. worker2, get 1_latest, and return


就算在update之前, 判断get回来的值的话, 会出现

1. worker2, set 1_latest value2

2. worker3, set 1_latest value3

3. worker1, set 1_latest value1

4. worker1, get 1_latest, and update

5. worker3, get 1_latest, and value1 < value3, update

6. worker2, get 1_latest, and value1 < value2, update

所以, worker3会被worker2给覆盖掉



