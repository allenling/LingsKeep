首先
======

1、InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。

2、由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。应用设计的时候要注意这一点。 

3、当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。 

4、即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。 

5、检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。

6. next_key锁是记录锁和gap组合的, 左开右闭的 是在索引上加锁，搜索使用主键就在主键上加，搜索通过二级索引就在二级索引上加.

  6.1 通过唯一索引, 比如primary key, unique key, 如果没找到，则锁上区间, 组成next_key锁, 如果有记录满足条件，则不会加上gap锁, 只有record锁.

  6.2 通过二级索引，并且是非唯一索引查找记录, 不管有没有记录满足条件, 都会加上gap锁组成next_key锁.
  
  6.4 这是为了解决幻读, 比如6.1中如果找到了一个唯一记录，那只需要加记录锁就好了，因为不会有新加的数跟唯一索引一样，比如id=1. 如果没有满足条件的唯一索引值，那么就需要上gap锁，否则其他事务加入一个id=1的记录就出现幻读了,
      6.2思路一样, 就算在6.2中找到了满足记录的索引，但是由于非唯一，select出来的个数可能会变化, 比如name=1有两个，这个时候事务2又添加一个name=1的话就出现幻读. 

  6.7 gap锁是可以共享的，前提是select for update的where不是同一个就好.

  6.8 有个特殊情况是，如果使用的是联合唯一索引中的一部分，不管有没有满足条件的记录，都会有gap锁. 这也是可以理解的，因为gap锁就是为了防止幻读的, 比如唯一联合(first_name, last_name), 然后
      select * from t where first_name='abc' for update, 并且存在记录('abc', 'd'), 如果不加上gap锁，那么有事务2插入('abc', 'e')就产生幻读了. Gap locking is not needed for statements that lock rows using a unique index to search for a unique row. (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.) 

  6.9 所以，分析gap的时候就要分析是否导致幻读. 包括范围搜索，比如id<5这样，如果插入或者删除id<5的数据就会出现幻读，所以这个时候也会有gap锁的.

7. 聚簇索引和非聚簇索引: http://wangxinchun.iteye.com/blog/2373650

8. 小表或者大小查询大部分数据的是, 索引不一定有好处, 有时候顺序读取所有数据可能比通过索引更好, 因为顺序索引有更小的磁盘寻址. https://dev.mysql.com/doc/refman/5.7/en/mysql-indexes.html: Indexes are less important for queries on small tables, or big tables where report queries process most or all of the rows. When a query needs to access most of the rows, reading sequentially is faster than working through an index. Sequential reads minimize disk seeks, even if not all the rows are needed for the query. See Section 8.2.1.19, “Avoiding Full Table Scans” for details.

9. 搜索二级索引的时候加上了x锁的话，mysql也会对主键(聚簇索引)上锁, 所以update/selct for update一个二级索引, 跟在主键上update/select for update没什么区别. https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html: If a secondary index is used in a search and index record locks to be set are exclusive, InnoDB also retrieves the corresponding clustered index records and sets locks on them.


10. 查询唯一索引/主键的时候，如果不存在，则会加上gap锁, 如果存在则只加记录锁，不加gap锁，如果存在但是被标识为删除状态，则会加gap锁, http://hedengcheng.com/?p=844. 这在删除的时候有可能会发生死锁

11. 各个加锁的情况: http://hedengcheng.com/?p=771, http://hedengcheng.com/?p=577 里面有关于index key, index filter, table filter以及icp的一些说明

12. 意向锁和插入意向锁?

    INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.

    Prior to inserting the row, a type of gap lock called an insert intention gap lock is set. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6 each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.

    **If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock** should there be multiple sessions trying to insert the same row if another session already has an exclusive lock. This can occur if another session deletes the row. Suppose that an InnoDB table t1 has the following structure:



select for update or insert
==============================

select for update, insert

当需要select for update or insert的时候，比如一个用户手否对某个文章有点赞记录，没有就创建，有的话就更新(点赞/取消赞), 这个时候在django中可能直接就用

model.objects.update_or_create(\**cond), 但是这样容易引发死锁. django的update_or_create是先select for update选中记录, 没有就insert, 这样容易引发死锁.

简单的来说就是select for update的时候的
**gap锁是可共享, 因为两者for update的条件又不冲突, 在文档https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-insert-intention-locks
上的那个图IX和IX共享就有点理解了, 这里的select for update是一个next_key锁, 然后包含两部分，key和gap, key不冲突，那么gap可以共享, 至少看起来是这样的
https://stackoverflow.com/questions/43827740/mysql-innodb-deadlock-between-select-for-update-and-insert** 

It is also worth noting here that conflicting locks can be held on a gap by different transactions. For example, transaction A can hold a shared gap lock (gap S-lock) on a gap while transaction B holds an exclusive gap lock (gap X-lock) on the same gap. The reason conflicting gap locks are allowed is that if a record is purged from an index, the gap locks held on the record by different transactions must be merged.

Gap locks in InnoDB are “purely inhibitive”, which means they only stop other transactions from inserting to the gap. They do not prevent different transactions from taking gap locks on the same gap. Thus, a gap X-lock has the same effect as a gap S-lock.


然后如果insert的时候是插入同一个区间的, 就发现t1的insert等待t2的select for update的gap锁释放, 然后t2的insert等待t1的select for update的gap锁释放.


参考: http://blog.csdn.net/zhwbqd/article/details/17056447, http://yeshaoting.cn/article/database/mysql%20insert%E9%94%81%E6%9C%BA%E5%88%B6/

解决的话只能再应用层先select, 然后判断再insert

或者insert into on duplicate key update


select for update/update
===========================

比如

	t1                                        t2
update t set name='t1' where id=1;         

                                           update t set name='t2' where id=2;



update t set name='1t2' where id=2;                        

                                          update t set name='2t1' where id=1;



t1的第二句要等待t2的第一句释放, t2的第二句要等待t1的第一句释放，所以死锁.

解法嘛就是加锁要按一定的顺序，比如先把id给排序号，再for update一下.

这种情况在使用异步任务同步redis数据到Mysql的时候有可能会发生, 比如你用scan这个及其不靠谱的命令来拿redis的key的时候.


concurrency insertion
========================

duplication dead lock, 并发插入的时候导致死锁，并且需要有3个或者3个以上的操作, 文档有详解
https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

t1, t2, t3分别都插入同样一个唯一索引, 比如id=1, 然后t1释放, 这时会造成死锁.

The first operation by session 1 acquires an exclusive lock for the row. The operations by sessions 2 and 3 both result in a duplicate-key error and they both request a shared lock for the row. When session 1 rolls back, it releases its exclusive lock on the row and the queued shared lock requests for sessions 2 and 3 are granted. At this point, sessions 2 and 3 deadlock: Neither can acquire an exclusive lock for the row because of the shared lock held by the other.

大意是t1 insert的时候，t2和t3都会因为duplicate-key error而降为加上S锁，然后t1 rollback之后，t2和t3会都申请X锁, 然后互相等待对方的S锁释放造成死锁.
但是没明白t1 insert的时候为什么t2和t3会发生duplicate error, 应该是被阻塞的呀.

这里说得明白点: http://www.cnblogs.com/sunss/p/3166550.html

大意是insert的时候需要先申请S锁检查是否冲突(这里的S锁应该是是insert intention gap lock, 插入意向间隙锁 Prior to inserting the row, a type of gap lock called an insert intention gap lock is set.), 然后
t1插入的时候先申请S锁，然后检查key是否冲突，发现没冲突，然后加上X锁，然后t2和t3要插入，这个时候也必须是申请insert intention gap lock这个S锁，但是被t1的X锁阻塞了, 然后t1 rollback, t2, t3都或得了S锁，检查
没有key冲突，然后申请X锁，这个时候t2等待t3的共享锁释放, t3等待t2的共享锁释放.


concurrency delete
====================

http://hedengcheng.com/?p=844

其实，以上两个加锁策略，都是正确的。以上两个策略，分别对应的是：1）唯一索引上满足查询条件的记录存在并且有效；2）唯一索引上满足查询条件的记录不存在。但是，除了这两个之外，其实还有第三种：3）唯一索引上满足查询条件的记录存在但是无效。众所周知，InnoDB上删除一条记录，并不是真正意义上的物理删除，而是将记录标识为删除状态。(注：这些标识为删除状态的记录，后续会由后台的Purge操作进行回收，物理删除。但是，删除状态的记录会在索引中存放一段时间。) 在RR隔离级别下，唯一索引上满足查询条件，但是却是删除记录，如何加锁？InnoDB在此处的处理策略与前两种策略均不相同，或者说是前两种策略的组合：对于满足条件的删除记录，InnoDB会在记录上加next key lock X(对记录本身加X锁，同时锁住记录前的GAP，防止新的满足条件的记录插入。) Unique查询，三种情况，对应三种加锁策略，总结如下：

找到满足条件的记录，并且记录有效，则对记录加X锁，No Gap锁(lock_mode X locks rec but not gap)；
找到满足条件的记录，但是记录无效(标识为删除的记录)，则对记录加next key锁(同时锁住记录本身，以及记录之前的Gap：lock_mode X);
未找到满足条件的记录，则对第一个不满足条件的记录加Gap锁，保证没有满足条件的记录插入(locks gap before rec)；


