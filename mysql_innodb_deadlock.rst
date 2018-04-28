首先
======

1. InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。

2. 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。应用设计的时候要注意这一点。 

3. 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。 

4. 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高

   比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。 

5. 检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。

6. next_key锁是记录锁和gap组合的, 左开右闭的 是在索引上加锁，搜索使用主键就在主键上加，搜索通过二级索引就在二级索引上加.

7. 通过唯一索引, 比如primary key, unique key, 如果没找到，则锁上区间, 组成next_key锁, 如果有记录满足条件，则不会加上gap锁, 只有record锁.

8. 通过二级索引，并且是非唯一索引查找记录, 不管有没有记录满足条件, 都会加上gap锁组成next_key锁.
  
9. 这是为了解决幻读, 比如找到了一个唯一记录，那只需要加记录锁就好了，因为不会有新加的数跟唯一索引一样，比如id=1.
   
   如果没有满足条件的唯一索引值，那么就需要上gap锁，否则其他事务加入一个id=1的记录就出现幻读了,

10. 如果找到了满足记录的索引，但是由于非唯一，select出来的个数可能会变化, 比如name=1有两个，这个时候事务2又添加一个name=1的话就出现幻读. 

11. 有个特殊情况是，如果使用的是联合唯一索引中的一部分，不管有没有满足条件的记录，都会有gap锁. 这也是可以理解的，因为gap锁就是为了防止幻读的, 比如唯一联合(first_name, last_name), 然后

    select * from t where first_name='abc' for update, 并且存在记录('abc', 'd'), 如果不加上gap锁，那么有事务2插入('abc', 'e')就产生幻读了.
    
    Gap locking is not needed for statements that lock rows using a unique index to search for a unique row.
    
    (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.) 

    以上来自文档: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

12. 所以，分析gap的时候就要分析是否导致幻读. 包括范围搜索，比如id<5这样，如果插入或者删除id<5的数据就会出现幻读，所以这个时候也会有gap锁的.

13. 聚簇索引和非聚簇索引: http://wangxinchun.iteye.com/blog/2373650

14. 小表或者大小查询大部分数据的是, 索引不一定有好处, 有时候顺序读取所有数据可能比通过索引更好, 因为顺序索引有更小的磁盘寻址.
    
    Indexes are less important for queries on small tables, or big tables where report queries process most or all of the rows. When a query needs to access most of the rows,
    
    reading sequentially is faster than working through an index. Sequential reads minimize disk seeks, even if not all the rows are needed for the query.
    
    See Section 8.2.1.19, “Avoiding Full Table Scans” for details.

    以上参考：　https://dev.mysql.com/doc/refman/5.7/en/mysql-indexes.html


15. 搜索二级索引的时候加上了x锁的话，mysql也会对主键(聚簇索引)上锁, 所以update/selct for update一个二级索引, 跟在主键上update/select for update没什么区别.
  
    if a secondary index is used in a search and index record locks to be set are exclusive, InnoDB also retrieves the corresponding clustered index records and sets locks on them.

    来自: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

16. 查询唯一索引/主键的时候，如果不存在，则会加上gap锁, 如果存在则只加记录锁，不加gap锁，如果存在但是被标识为删除状态，则会加gap锁, http://hedengcheng.com/?p=844. 这在删除的时候有可能会发生死锁

17. 各个加锁的情况: http://hedengcheng.com/?p=771, http://hedengcheng.com/?p=577 里面有关于index key, index filter, table filter以及icp的一些说明


死锁案例
=============

死锁分析关键点在于一般都是gap锁导致的, 一般在RR隔离级别下分析加锁的时候, 关键点在于分析解幻读的情况, 也就是:

1. 什么索引, 唯一索引还是二级索引
   
2. gap锁怎么设置的

意向锁/插入意向锁死锁
=========================

参考: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-insert-intention-locks

An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion, a type of gap lock called an insert intention gap lock is set.

This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for

each other if they are not inserting at the same position within the gap.

Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6 each lock the

gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.

插入的时候会在 **插入之前**, 设置一个叫插入意向锁, gap类型的锁, 但是如果插入的不是同一条记录但是是同一个gap的话, 并不会阻塞.

在4, 7之间插入5, 6并不会互相阻塞对方


下面参考: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock)

and does not prevent other sessions from inserting into the gap before the inserted row.

插入的时候在记录上加的锁不是next-key锁, 而只是单纯的记录锁, 但是插入之前的意向锁会和互斥的gap锁互斥, 比如

事务1中, select for update, 锁了一个gap, 当插入这个gap的时候, 插入意向锁和互斥锁互斥, 所以插入被阻塞

**注意: 前面说插入的时候不加gap锁是说记录上不是next-key锁, 只是一个记录锁, 而后面说的插入意向gap锁, 是插入之前加的锁**

插入时加入意向锁和互斥锁
===============================

**所以, 插入的时候, 有两个锁, 先插入意向锁, 然后单个记录上的互斥锁!!!!**

下面的例子说明:

.. code-block:: python

    '''

    name是一个非唯一的二级索引
    
    mysql> select * from t1;
    +----+------+
    | id | name |
    +----+------+
    |  1 | t1   |
    |  3 | t11  |
    |  4 | t110 |
    |  2 | t2   |
    +----+------+
    4 rows in set (0.00 sec)
    
    '''

然后:

.. code-block:: python

    '''
    
    事务1:
    
    mysql> INSERT INTO t1(name) VALUES('t111');
    Query OK, 1 row affected (0.00 sec)
    
    事务2:
    
    mysql> INSERT INTO t1(name) VALUES('t112');
    Query OK, 1 row affected (0.00 sec)
    
    '''

此时插入虽然是同一个区间(t110, t2]之间, 但是插入是不会阻塞的, 插入意向锁可以共享, 然后事务2进行select for update

.. code-block:: python

    '''
    
    select * from t1 where name>'t110' for update;
    
    '''

此时事务2被阻塞, 也就是插入意向锁阻塞了select for update的互斥锁, 证明了插入意向锁的存在

然后继续, 证明插入的时候, 会在插入记录上加互斥锁:

.. code-block:: python

'''

mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | t1   |
|  3 | t11  |
|  4 | t110 |
|  5 | t111 |
|  6 | t112 |
|  2 | t2   |
+----+------+
6 rows in set (0.00 sec)

事务1:

INSERT INTO t1(name) VALUES('t113');

事务2:

INSERT INTO t1(name) VALUES('t114');

'''

然后事务2对name=t113进行去select for update

.. code-block:: python

    '''
    
    事务2: select * from t1 where name='t113' for update;
    
    '''

此时阻塞

duplicate error导致死锁
==========================

**原因是, duplicate error会导致在索引上加上共享锁, 要注意下如何会导致duplicate error**

下面参考: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should

there be multiple sessions trying to insert the same row if another session already has an exclusive lock.

This can occur if another session deletes the row. Suppose that an InnoDB table t1 has the following structure:

.. code-block:: python

    '''
    CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
    '''

Now suppose that three sessions perform the following operations in order:

.. code-block:: python

    '''
    
    session 1:
    
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    
    session 2:
    
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    
    session 3:
    
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    
    '''

session 1 rollback

.. code-block:: python

    '''
    
    ROLLBACK;
    '''

The first operation by session 1 acquires an exclusive lock for the row. The operations by sessions 2 and 3 both result in

a duplicate-key error and they both request a shared lock for the row. When session 1 rolls back,

it releases its exclusive lock on the row and the queued shared lock requests for sessions 2 and 3 are granted.

At this point, sessions 2 and 3 deadlock: Neither can acquire an exclusive lock for the row because of the shared lock held by the other.

插入的时候如果产生duplicate-key错误, 那么一个共享锁会设置在产生冲突的索引上. **注意这里, 共享锁加到有冲突的索引上**

所以此时插入会出现死锁. 文档中的例子就是三个事务分别插入同一个记录, 然后事务1回退, 那么事务2和事务3发送死锁

所以, 死锁是由于2, 3都请求成功共享锁, 然后插入的时候需要申请互斥锁, 此时2的插入的互斥锁等待3的共享锁释放, 3也一样:

1. 事务1插入成功, 在记录上设置互斥锁

2. 事务2, 3请求插入的时候发现记录存在(因为互斥锁), 所以发送duplicate error, 然后

   事务2, 3都请求索引的共享锁

3. 事务1回退, 那么事务2, 3的共享锁都申请成功, 但是又插入的时候, 事务2需要加上一个互斥锁, 必须等待事务3释放共享锁

   同样事务3也一样, 造成死锁

三个插入一个提交死锁
========================

下面参考: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

.. code-block:: python

    '''
    
    session 1:
    
    START TRANSACTION;
    DELETE FROM t1 WHERE i = 1;
    
    session 2:
    
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    
    session 3:
    
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    
    '''

session 1 commit

.. code-block:: python

    '''
    
    commit;
    '''

同样是死锁, 也就是事务1执行删除操作的时候, 加了互斥锁, 然后事务2和3都产生duplicate error, 然后流程是rollback的时候一样了


**所以, 插入导致死锁是因为产生duplicate error的时候, 转而在索引上请求共享锁, 然后多事务再次插入的时候和共享锁产生死锁**

注意分析duplicate error的产生, 以及共享锁的产生

select for update然后insert死锁
==================================

这里发送死锁是因为select for update的时候, 搜索不到, 对同一个区间加入了gap锁, 此时两个gap锁不冲突.

因为虽然锁的区间一样, 但是select for update的时候, where的条件, 也就是索引不一样, 所以不冲突

然后插入的时候, 互相等待select for update的gap释放

比如50, 80, 分别select for update, 第一个where是60, 第二个where是70, 两个事务都锁住了(50, 80]的区间, 但是不互斥

然后分别插入60和70, 插入意向锁互相导致对方的gap释放, 导致死锁

解决的话只能再应用层先select, 然后判断再insert

或者insert into on duplicate key update


select for update然后update死锁
=================================

顺序不同互相等待导致死锁

比如

.. code-block:: python

    '''
    
    	t1                                        t2
    update t set name='t1' where id=1;         
    
                                               update t set name='t2' where id=2;
    
    
    
    update t set name='1t2' where id=2;                        
    
                                              update t set name='2t1' where id=1;
    
    '''

t1的第二句要等待t2的第一句释放, t2的第二句要等待t1的第一句释放，所以死锁.

解法嘛就是加锁要按一定的顺序，比如先把id给排序号，再for update一下.

这种情况在使用异步任务同步redis数据到Mysql的时候有可能会发生, 比如你用scan这个及其不靠谱的命令来拿redis的key的时候.


三个删除一个提交死锁
=====================

这个和上面说的意向锁的时候的删除死锁的情况是不一样的, 上面的是一个删除导致后面的两个事务死锁, 这个例子是

三个事务同时删除唯一索引下的记录导致的死锁

http://hedengcheng.com/?p=844

**Unique查询，三种情况，对应三种加锁策略，总结如下:**

1. 找到满足条件的记录，并且记录有效，则对记录加X锁，No Gap锁(lock_mode X locks rec but not gap)；

2. 找到满足条件的记录，但是记录无效(标识为删除的记录)，则对记录加next key锁(同时锁住记录本身，以及记录之前的Gap：lock_mode X);

3. 未找到满足条件的记录，则对第一个不满足条件的记录加Gap锁，保证没有满足条件的记录插入(locks gap before rec)；


