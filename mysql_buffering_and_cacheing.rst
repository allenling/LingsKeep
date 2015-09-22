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

**query cache在数据库数据比较静态, 变动不是很经常的情况下非常有用.**

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

*而nagios的check_mysql_health中的qcache-hitrate的计算方法是Qcache_hits/(Qcache_hits + Com_select)*


Innodb Buffer Pool [#]_
------------------------

innodb在内存中会维护一个称为buffer pool的存储区域来缓存数据和索引.

What is Innodb Buffer Pool
~~~~~~~~~~~~~~~~~~~~~~~~~~~

innodb buffer pool的大小越大, innodb越表现得像一个内存数据库. buffer pool也缓存被insert和update更改过的数据, 而不像query cache只缓存select语句.

在64位系统中, 你可以把很大的buffer pool分隔成多个部分以最小化并发产生的内存争用(to minimize contention for the memory structures among concurrent operations)

How Innodb Buffer Pool works
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

innodb buffer pool是\ **LRU(Least Recently Used)**\ 算法的一个实现.

innodb buffer pool是一个列表, 需要缓存的block会被插入到这个列表中.当列表被填满并且有新的block要添加到列表中的时候, innodb会将一个处于列表最后一位的block剔除出
buffer pool的列表中. 新的block插入到列表中的\ **中间位置**, 称为中间插入策略(midpoint insertion strategy). 这样就把buffer pool列表\ **形式**\ 上分成两个子列表:

1. new列表, 以中间位置为基准, 越往列表头部的block是越新的.
2. old列表, 以中间位置为基准, 越往列表尾部的block是越旧的.

buffer pool的结构是这样的

[new blocks, new inserted block, old blocks]

**这样中间位置是old列表的第一位的前一位**

* innodb会将最经常用的block放在new列表中, 而将不经常使用的block放在old列表中, 在old列表中最后一个block会被剔除出列表当整个buffer pool已被填满但又有新的block需要插入的时候.

* 在buffer pool中, 默认3/8大小的大小是用来存储old列表的.

* 当用户请求一个query, 或者innodb自动的read-ahead操作的时候, 产生的block就会被插入到buffer pool中.

* 一旦一个在old列表中的block被访问, 则innodb会将这block往前移到buffer pool的第一位(new列表的第一位), 使之变为'young'. 如果这个block被访问是因为用户请求的, 则innodb会马上返回
  block, 并且立刻将block移动到列表第一位, 若是因为innodb的read-ahead请求的, 则innodb并不会立马返回并移动block到列表的第一位.

* 这样, 一旦有block\ **(无论是在new列表还是old列表)**\ 被访问, 它总会被移动到new列表的第一位, 而一直没有被访问的block就自动地往buffer pool尾部移动, 这样当需要剔除block的时候, 直接剔除列表最后一个block就可以了.


值得注意的是, 一个全表查询(mysqldump操作, 不带where的select语句, 一些性能测试等)会产生大量的block插入到buffer pool中, 这样可能导致大量的block被剔除, 或者大量的block移动到old列表中.
而这些查询的block只使用一次的情况下很明显对innodb的性能有比较大的影响, 这个情况下可以通过调整下面innodb_old_block_time数值来避免.


Innodb Buffer Pool Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* innodb_buffer_pool_size
  buffer pool的大小

* innodb_buffer_pool_instance
  buffer pool的实例个数, 这个配置只在buffer pool的大小至少大于1G的时候才有效. 这个配置会将buffer pool分隔成多个独立的区域, 这样可以减少并发下的内存读写的争用.

* innodb_old_block_pct
  old列表的大小百分比, 范围是5-95, 默认是37(3/8).

* innodb_old_block_time
  在old列表中的block\ **能**\ 被移动到new列表之前, 它在old列表必须存在的毫秒数, 默认是0. 若该值大于0, 比如1000, 也就是, 一个在old列表的block在第一次访问之后, 并不会立刻被移动到new列表中
  而是继续存在old列表中, 直到这1秒之后有一个访问, 这个时候, 这个block才会被移动到new列表中.

Setting innodb_old_blocks_time greater than 0 prevents one-time table scans from flooding the new sublist with blocks used only for the scan. Rows in a block read in for a scan are
accessed many times in rapid succession, but the block is unused after that. If innodb_old_blocks_time is set to a value greater than time to process the block, the block remains in
the “old” sublist and ages to the tail of the list to be evicted quickly. This way, blocks used only for a one-time scan do not act to the detriment of heavily used blocks in the new
sublist.

设置innodb_old_block_time默认值为0, 这样一次性的全表查询, 例如有些性能测试, 使得大量处于old列表的block移动到new列表中, 而原先new列表中的block就自动地移动到buffer pool的尾部,
而这样一旦需要剔除buffer pool的block, 原先new列表中的block就会被优先剔除了. 若设置一个大于0的值(set to a value greater than time to process the block), 则原先old列表中的block
则不会马上移动到new列表, 这样就不会危害到new列表中的blocks了.

可以在运行时设置innodb_old_block_time, 所以你可以在性能测试期间暂时地将innodb_old_block_time增大, 之后在将innodb_old_block_time设置为0.

Monitoring and Buffer Pool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SHOW ENGINE INNODB STATUS输出的innodb状态中, 在BUFFER POOL AND MEMORY一节输出了buffer pool的状态.

.. code-block:: python

    mysql> SHOW ENGINE INNODB STATUS;
    BUFFER POOL AND MEMORY
    ----------------------
    Total memory allocated 536576000; in additional pool allocated 0
    Dictionary memory allocated 2311279
    Buffer pool size   31999
    Free buffers       26133
    Database pages     5834
    Old database pages 2171
    Modified db pages  0
    Pending reads 0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 5825, created 9, written 318
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 5834, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]


其中

* Old database pages: old列表中的page数量

* Pages made young, not young: 从old列表中移动到new列表中的page数量, old列表中的从没有被移动到new列表的page数量.

* youngs/s, non-youngs/s: 访问old page中的page使得page移动到new列表的访问个数, 多个访问同一个old page, 则依然会计数. non-young/s跟youngs/s相反.

* young-making rate: 缓存命中中使得block移动到new列表的的比率.

* not: 缓存命中中, 但是不满足innodb_old_block_time的要求而没有使得block移动到new列表的的比率.

当没有大量的查询产生但是youngs/s值却很低的时候(一般查询应该尽可能保持block处于new列表中, 尽量不优先被剔除, youngs/s应该比较大), 可以要么减少innodb_old_block_time的值, 要么调高
old列表在buffer pool的大小百分比.
调高old列表大小的百分比会延缓block到达buffer pool最后一位并被剔除的时间.

如果产生大量的查询, 但是non-youngs/s并没有相应的增大, 则可以将innodb_old_block_time增大.
全表查询应尽可能处于old列表中, 防止全表查询产生的block只访问一次而导致之前很多new列表中的block变得old.

**也就是说**:

1. 大量查询下, non-youngs/s应该变大, 增大innodb_old_block_time大小.

2. 一般情况下, youngs/s不应该低, 减小innodb_old_block_time的大小或调大old列表的大小.

.. [#] https://dev.mysql.com/doc/refman/5.5/en/buffering-caching.html
.. [#] https://dev.mysql.com/doc/refman/5.5/en/query-cache.html
.. [#] https://dev.mysql.com/doc/refman/5.5/en/innodb-buffer-pool.html

