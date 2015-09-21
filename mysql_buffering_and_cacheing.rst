MySQL Buffering and Caching [#]_
==================================

MySQL的caching包括

* The InnoDB Buffer Pool
* The MyISAM Key Cache
* The MySQL Query Cache

Query Cache [#]_
-----------------

What is query cache
~~~~~~~~~~~~~~~~~~~~

mysql会对select语句以及select语句的结果进行缓存, 一旦命中缓存, mysql直接从query cache中获取结果 返回给客户端. query cache是多会话(session)共享的, 一个会话缓存了一条select语句,
而另外一个会话请求了同样一条select语句, 则mysql依然会从query cache中返回结果.

任何对表的修改, 包括update, delet等, 都会把query cache中对应表的缓存给清除(flush)掉.

query cache在数据库数据比较静态, 变动不是很经常的情况下非常有用.

query cache也会有维护的代价(内存分配等), 一般而言, query cache的命中概率越大越好, 若命中率在50%一下, 基本上可以关闭query cache了.

而是否开启query cache需要依据业务来决定, 而query cache的配置值也需要根据业务来调整.

How the Query Cache Operates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

query cache缓存命中的条件是两个select语句一模一样(exactly the same, byte for byte), 语句中使用不同的数据库, 不同的协议版本, 不同的字符集等都会被认为是不想同的两个语句.

query cache并不会对一下的语句缓存:

1. Queries that are a subquery of an outer query
2. Queries executed within the body of a stored function, trigger, or event

命中缓存后, mysql不会因为没有解析和执行语句而不校验权限, 而是依然会对权限进行校验.在mysql从query cache获取结果之前, mysql会检查用户是否对语句中所有的数据库, 表有select权限.

一旦命中缓存, 并且返回了缓存结果, 则mysql会增加Query hits的值, 而不是Com_select的值(变量含义在下面Query Cache Status and Maintenance一节).

一旦一个表有变化, 则该表在query cache中的所有关联到该表的缓存都会被清除, 表的变化包括使用MEGE来合并表, insert, update, delete, truncate table, alter table, drop table, 或者drop database.

query cache在innodb的事务中同样可以使用.

若语句中出现包含在下面表中任何一个函数, query cache并不会缓存语句.

============================ ================= ===================================
BENCHMARK()                  CONNECTION_ID()   CONVERT_TZ()
CURDATE()                    CURRENT_DATE()    CURRENT_TIME()
CURRENT_TIMESTAMP()          CURTIME()         DATABASE()
ENCRYPT() with one parameter FOUND_ROWS()      GET_LOCK()
LAST_INSERT_ID()             LOAD_FILE()       MASTER_POS_WAIT()
NOW()                        RAND()            RELEASE_LOCK()
SLEEP()                      SYSDATE()         UNIX_TIMESTAMP() with no parameters
USER()                       UUID()            UUID_SHORT()
============================ ================= ===================================

若一个select语句有下列任一一种情况, query cache也不会生效:

* 关联到用户自定义函数
* 关联的用户变量或本地存储的编程变量
* 关联到mysql, INFORMATION_SCHEMA, 或者performance_schema数据库
* MySQL5.5.23之后, 关联到分区表(refers to any partitioned tables)
* 下列任一形式:
  - SELECT ... LOCK IN SHARE MODE
  - SELECT ... FOR UPDATE
  - SELECT ... INTO OUTFILE ...
  - SELECT ... INTO DUMPFILE ...
  - SELECT * FROM ... WHERE autoincrement_col IS NULL
* 使用临时表
* 不使用任何表
* 出现warnings
* 用户对任一关联的表有一个列级别的权限(The user has a column-level privilege for any of the involved tables.)

Query Cache SELECT Options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. 在select语句中可以使用query cache, 使用SQL_CACHE.

SELECT SQL_CACHE id, name FROM customer;

2. 在select中不使用query cache, 使用SQL_NO_CACHE

SELECT SQL_NO_CACHE id, name FROM customer;

Query Cache Configuration And Query Cache Status and Maintenance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

变量have_query_cache显示是否已经开启了query cache

.. code-block:: mysql

    mysql> SHOW VARIABLES LIKE 'have_query_cache';
    +------------------+-------+
    | Variable_name    | Value |
    +------------------+-------+
    | have_query_cache | YES   |
    +------------------+-------+

但是, **When using a standard MySQL binary, this value is always YES, even if query caching is disabled.**

query cache主要是使用query_cache_%变量来来控制, 如:

.. code-block:: mysql

    mysql> SHOW VARIABLES LIKE 'query_cache_%';
    +------------------------------+----------+
    | Variable_name                | Value    |
    +------------------------------+----------+
    | query_cache_limit            | 1048576  |
    | query_cache_min_res_unit     | 4096     |
    | query_cache_size             | 16777216 |
    | query_cache_type             | ON       |
    | query_cache_wlock_invalidate | OFF      |
    +------------------------------+----------+

quer_cache_size表示query cache的内存大小, 可以设置为0, 表示禁用query cache. 当query_cache_size设置为0的时候, query_cache_type则会影响query cache如何起作用.

query_cache_type表示query cache的行为类型:

* 0或者OFF表示不会不会缓存和检索query cache(prevents caching or retrieval of cached results)
* 1或者ON表示只有select语句明确 **不** 需要缓存(使用 SQL_NO_CACHE)的时候才 **不** 进行缓存
* 2或者DEMAND表示只有当select明确需要缓存(使用SQL_CACHE)的时候才缓存

当不使用query cache的时候, 设置query_cache_size为0, 同时为了将quer cache的开销显著减少, 将query_cache_type也设置为0.
(To reduce overhead significantly, also start the server with query_cache_type=0 if you will not be using the query cache)

query_cache_size最小值为4kb, 小于40kb则会有warning, 不建议将query_cache_size设置过大, 这是因为更新缓存的时候, 线程会锁住缓存, 过大的缓存会引发锁竞争问题(
Be careful not to set the size of the cache too large. Due to the need for threads to lock the cache during updates, you may see lock contention issues with a very large cache.)

query_cache_limit是单个缓存的大小, 默认为1M.

缓存除了缓存语句之外, 还会缓存语句的结果, 一般结果跟语句是分开存储, mysql会分配块(blocks)来存储结果.query_cache_min_res_unit是设置块(block)的最小大小.

query_cache_min_res_unit默认值为4KB, 这个值足以应付绝大多数情景.

如果你的query的结果的大小几乎都是很小的, 默认的query_cache_min_res_unit值可能会导致内存碎片, 这是因为存储都是以block为单位的, 也就是一个block只存储了一个很小的返回结果
(大概类似于block为4KB, 而一个query的结果只有4B), 这样, query cache会有大量的未使用大小. 而当query_cache_size不足的时候, query cache会删除一部分缓存来释放内存. 所以这个时候可以适当地
降低query_cache_min_res_unit的值. 想法的情况可以适当地增大block的大小. 可以使用flush query cache命令来对query cache进行碎片整理, 该命令不会删除缓存, 而reset query cache则是重置
query cache, 意味是删除query cache中所有的缓存.

Qcache_free_blocks表示query cache中可用的block个数

Qcache_lowmem_prunes表示由于内存不足被从query cache中删除的block个数.

.. code-block:: mysql

    mysql> show status like 'Qcache%';
    +-------------------------+----------+
    | Variable_name           | Value    |
    +-------------------------+----------+
    | Qcache_free_blocks      | 12       |
    | Qcache_free_memory      | 16120224 |
    | Qcache_hits             | 9897     |
    | Qcache_inserts          | 6325     |
    | Qcache_lowmem_prunes    | 0        |
    | Qcache_not_cached       | 257      |
    | Qcache_queries_in_cache | 416      |
    | Qcache_total_blocks     | 914      |
    +-------------------------+----------+

之前提到, 当命中缓存的时候, 会增加Qcache_hits的值, 而不是Com_select的值, Qcache_inserts则是未命中缓存并且被添加到query cache的语句个数.

Qcache_free_memory则是query cache中可用内存, Qcache_not_cached则是为被缓存的语句个数. 其他的可以顾名思义.

而Com_xxx开头的变量表示xxx的个数, Com_select就是select语句的个数

Com_select = Qcache_hits + queries with errors found by parser

Qcache_inserts = Qcache_not_cached + queries with errors found during the column-privileges check

而nagios的check_mysql_health中的qcache-hitrate的计算方法是Qcache_hits/(Qcache_hits + Com_select)


Innodb Buffer Pool [#]_
------------------------

.. [#] https://dev.mysql.com/doc/refman/5.5/en/buffering-caching.html
.. [#] https://dev.mysql.com/doc/refman/5.5/en/query-cache.html
.. [#] https://dev.mysql.com/doc/refman/5.5/en/innodb-buffer-pool.html

