RR和RC隔离级别加锁的区别
=========================

http://tech.meituan.com/innodb-lock.html
http://hedengcheng.com/?p=771


在RR隔离级别下，mysql会把扫描但是不满足where条件的row也给锁住， 而RC缺不会, RR会有gap锁，和行锁组成next-key锁, 而RC没有gap锁. 

gap只会出现在RR级别下where条件是非唯一索引和没有索引的情况


例子(来自最后mysql的文档), a,b两列, 事务均set autocommit=0, update语句是使用exclusive lock，互斥锁:

1. 先插入数据

INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);

2. 第一个事务

UPDATE t SET b = 5 WHERE b = 3;

3. 第二个事务

UPDATE t SET b = 4 WHERE b = 2;


由于没有索引，则mysql做全表扫描.

4. 在RR级别下

第一个事务加锁情况:

x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock

第二个事务扫描到(1,2)这一行的时候，会等待事务1释放锁:

x-lock(1,2); block and wait for first UPDATE to commit or roll back


5. 在RC级别下

第一个事务加锁:

对于(1,2)，mysql扫描加锁，然后发现不符合where条件，释放掉.

x-lock(1,2); unlock(1,2)
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); unlock(3,2)
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); unlock(5,2)

第二个事务的加锁情况:

x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock


第二个事务读取(1,2)这一行的时候，会使用一个叫semi-consistent读取的方式

semi-consistent read是read committed与consistent read两者的结合。一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)。
semi-consistent read只会发生在read committed隔离级别下，或者是参数innodb_locks_unsafe_for_binlog被设置为true。

(以上semi-consistent的句子来自:http://hedengcheng.com/?p=220)

https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html


read view
============

MVCC都是基于version的.


RR和RC读取数据的时候，read view有区别.

In REPEATBLE READ, a ‘read view’ ( trx_no does not see trx_id >= ABC, sees < ABB ) is created at the start of the transaction, and this read view (consistent snapshot in Oracle terms) is held open for the duration of the transaction.

In READ COMMITTED, a read view is created at the start of each statement.

https://www.percona.com/blog/2012/08/28/differences-between-read-committed-and-repeatable-read-transaction-isolation-levels/


关于snapshot version:

What happens instead is – when transaction is started, the list of concurrently running transactions (not committed) is memorized.

https://www.percona.com/blog/2007/12/19/mvcc-transaction-ids-log-sequence-numbers-and-snapshots/


综合上述两个说法，RR一开始的时候就记住了所以活跃但是位提交的事务列表，并且保存起来知道事务提交，这就是read view, 然后每次query都是基于这个read view来做版本号过滤的.

RC每次query都会生成一个新的read view，然后根据version号以及是否提交来过滤.





