索引的Selectivity(选择度)/Cardinality(基数)
==============================================

索引的评估是由Selectivity和Cardinality来决定的.

1. Cardinality(基数)
---------------------

Cardinality: In SQL, cardinality refers to the number of unique values in particular column.表示该列有多少个可能的唯一值, 比如这一列, 只能填male/female, 则Cardinality就是2.

mysql中, show index from tbl_name显示tbl_name的Cardinality, 执行analyze tbl_name会更新Cardinality.

An estimate of the number of unique values in the index. This is updated by running ANALYZE TABLE or myisamchk -a. Cardinality is counted based on statistics stored as integers, so the value is not necessarily exact even for small tables. The higher the cardinality, the greater the chance that MySQL uses the index when doing joins.

2. Selectivity(选择度). 基数在总数中的百分比
-----------------------------------------------

Selectivity of index = cardinality/(number of records) * 100%. number of records就是表中的行数.

一个有100000行的表中, 性别这一列的Cardinality为2, 则Selectivity = 2/10000 * 100% = 0.02%

数据库的query optimizers会根据Selectivity来决定是否使用该索引.

索引的Selectivity越大, query optimizers在搜索的时候有越大的几率会用到该索引.

这是因为有些情况(通常是很小的Cardinality, Selectivity和Cardinality正相关, 所以也就是很小的Selectivity)访问索引不如全表扫描, 并且访问索引也是消耗资源的, 并且有时候会更慢.

例如, 在10000列的表中, 一半是男性,一半是女性. 我们需要找到所有女性的名字, 若用性别这一个索引, 则需要访问5000次索引才能找到所有的女性, 而全表扫描的话, 每一行有50%的几率为女性.

所以, 即使某一列定义了索引, 但是在搜索中不一定会用到.

sql语句中的user index并不会一定使用指定的索引, 这个只是建议而已, 索引使用取决与query optimizers.

索引和非索引搜索区别在于线性搜索和二分搜索的效率区别.前者平局N/2, 后者log2N. 而创建索引的代价是资源的开销.

如果where的类型不对，那么mysql不会用索引的, 比如name这一列搜索的时候where name=1是用不到索引的, 而where name='1'是可以用到索引的.
并且

.. [#] http://www.programmerinterview.com/index.php/database-sql/selectivity-in-sql-databases/
.. [#] http://stackoverflow.com/questions/1108/how-does-database-indexing-work
.. [#] http://dev.mysql.com/doc/refman/5.5/en/mysql-indexes.html
.. [#] http://sqlmag.com/database-performance-tuning/which-faster-index-access-or-table-scan


2. 聚簇索引/非聚簇索引
===========================


聚簇索引是一组b+树，然后叶节点存储了行数据，而非聚簇索引叶节点存储的是行数的指针，显然，聚簇索引在io上比非聚簇索引更优, 虽然空间更大一点.


二级索引存储的是索引行的数据和主键, 比如name是二级索引，索引树上除了存储name的数据，也存储了对应的主键数据.


3. explain的一些输出解析
=================================



假设uniq是表tas的一个唯一索引, unqi=500并不存在, 如果我们

   mysql> explain select * from tas where uniq=500;
   +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
   | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                          |
   +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
   |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | no matching row in const table |
   +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
   1 row in set, 1 warning (0.00 sec)

explain出来的结果都是NULL, 并且Extra中是no matching row in const table, 这个extra的意思是: For a query with a join, there was an empty table or a table with no rows satisfying a unique index condition.

type为const表示该表只可能有0或者1个满足条件(most one match row)数据, 这个时候一般是使用了主键或者唯一索引的时候才可能, 毕竟主键和唯一索引是唯一的，并且只有一条等于或者没有.

如果表只有一条数据的时候type为system, system是const的特殊情况.
    
https://www.percona.com/blog/2010/06/15/explain-extended-can-tell-you-all-kinds-of-interesting-things/
You might notice a few odd things about this EXPLAIN. First, there are no tables listed. Taking a look at the Extra column we see that MySQL mentions ‘const’ tables. A ‘const’ table is a table that contains 0 or 1 rows, or a table on which all parts of a primary key or unique key lookup are satisfied in the where clause. If a ‘const’ table contains no rows, and it is not used in an OUTER JOIN, then MySQL can immediately return an empty set because it infers that there is no way that rows could be returned. MySQL does this by adding the WHERE clause in the query with ‘where 0’.

所以说一个查询中, 如果where使用了主键或者唯一索引，并且记录不存在或者空表, 那么mysql可以立即(immediately)推断(infers)记录不存在, 并且返回不存在 

可以看到有个warning, 输入show warnings, 输出的最后一列会带有where 0的标识，表示肯定没有满足的数据，所以where 0, 如果表只有一行(不是只有一行满足条件), 那么mysql会直接读取该
行, 然后再做query plan, 并且show warnings输出的最后一列是where 1.

如果没用到索引, 那么type的值就是ALL, 表示读取全表, 然后如果有需要加锁(比如update)的话是先加锁, 然后发现不满足条件, 解锁(**待考察**).

eq_ref和ref都和join有关系:

如果用到了索引，并且join的是primary key/unique的话，就是eq_ref

.. code-block:: 

    mysql> explain select *  from tas,my where tas.id=my.id;
    +----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+
    | id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref           | rows | filtered | Extra |
    +----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+
    |  1 | SIMPLE      | my    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL          |    3 |   100.00 | NULL  |
    |  1 | SIMPLE      | tas   | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | testpro.my.id |    1 |   100.00 | NULL  |
    +----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+


如果用到索引了, 但是索引只是复合索引的最左前缀的一部分或者索引不是primary key/unqiue key, 则type就是ref.

.. code-block:: 

    mysql> explain select * from tas where name = '40';
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | tas   | NULL       | ref  | name_index    | name_index | 30      | const |    1 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)


如果用到了索引，并且是非等于查询 type为range, 比如<>, >, in, between这种: Only rows that are in a given range are retrieved, using an index to select the rows. The key column in the output row indicates which index is used. The key_len contains the longest key part that was used. The ref column is NULL for this type. range can be used when a key column is compared to a constant using any of the =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, or IN() operators:

一般show warnings会出现mysql改写sql语句的样子, 比如

.. code-block::

    mysql> explain select * from tas where name='sad';
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | tas   | NULL       | ref  | name_index    | name_index | 30      | const |    1 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)


    mysql> show warnings;
    +-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Level | Code | Message                                                                                                                                                                                                         |
    +-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Note  | 1003 | /* select#1 */ select `testpro`.`tas`.`id` AS `id`,`testpro`.`tas`.`name` AS `name`,`testpro`.`tas`.`uniq` AS `uniq`,`testpro`.`tas`.`tag` AS `tag` from `testpro`.`tas` where (`testpro`.`tas`.`name` = 'sad') |
    +-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

然后就看到Message里面的写法, 是不是很像一些orm翻译之后的sql语句, 比如django和peewee, 我之前还纳闷为什么这么写的, 原来是mysql觉得这样比较优.

rows表示Mysql预估有多少行需要检查, 一般是通过索引去预估有多少行需要等待检查, rows应该跟index key, index filter, table filter有关: http://hedengcheng.com/?p=577.

The rows column indicates the number of rows MySQL believes it must examine to execute the query.
For InnoDB tables, this number is an estimate, and may not always be exact.

参考https://segmentfault.com/q/1010000004532402:
这个rows就是mysql认为必须要逐行去检查和判断的记录的条数。 
举个例子来说，假如有一个语句 select * from t where column_a = 1 and column_b = 2;
全表假设有100条记录，column_a字段有索引（非联合索引），column_b没有索引。
column_a = 1 的记录有20条， column_a = 1 and column_b = 2 的记录有5条。

那么最终查询结果应该显示5条记录。 explain结果中的rows应该是20. 因为这20条记录mysql引擎必须逐行检查是否满足where条件。


然后filted:
The filtered column indicates an estimated percentage of table rows that will be filtered by the table condition. That is, rows shows the estimated number of rows examined and rows × filtered / 100 shows the number of rows that will be joined with previous tables.

关于rows和filtered: https://dba.stackexchange.com/questions/164251/what-is-the-meaning-of-filtered-in-mysql-explain


extra里面using index, using where, using index condition的一些区别, 关于mysql icp(Index Condition Pushdown Optimization):

https://segmentfault.com/q/1010000004197413

using index表示你使用了只需要过滤一个索引树就可以得出满足条件的记录. 比如select name from t where name = 'abc', 这样直接扫描索引树就ok了,因为二级索引name上也包含了
name的信息. 如果是select name from t where name > 'abc', 则explain的extra中就是using index;using where, 并且rows大于1，说明还需要对row过滤, 具体为什么，不太明白.

using index condition 跟mysql的icp有关: https://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html
icp的话大概就是首先查询的时候一般有mysql server和storage engine两个角色, 一般都是storage engine通过索引返回数据, 这里的数据是全列数据，然后mysql server再这些数据的基础上

做where过滤，而icp中, 如果where中的条件包含了索引，那么Mysql server会把where条件也发送到storage engine, 由storage engine根据索引来做一部分过滤, 这样在storage engine上就减少了返回的数据.

icp的限制为: innodb的二级索引, type=range, ref, eq_ref和ref_or_null的query, 显然主键上的icp没有什么效果, 因为icp的目的是减少全行读取来减少io, 而主键上就自己带有全行数据了.
For InnoDB tables, ICP is used only for secondary indexes. The goal of ICP is to reduce the number of full-row reads and thereby reduce I/O operations. For InnoDB clustered indexes, the complete record is already read into
the InnoDB buffer. Using ICP in this case does not reduce I/O.

using where表示先使用了索引拿到rows, 再应用where里面的条件去进行过滤, 比如name是一个二级索引, name > 'sad' and tag='1', 通过索引name查询到数据之后，还要根据查询到的数据查询tag='a'的数据. 

using index condition总是跟需要的不仅仅是单个索引树的信息有关, 比如你select * from t where name>'abc'的时候，需要全行数据，但是可以根据索引name来优先过滤出行数，所以会显示using index condition,
但是如果你select name,tag from t where name > 'a', 其中tag不是在二级索引name中, 所以自然也是using index condition, 其他的比如一个复合索引(first_name, last_name), 如果你select first_name, last_name的话, 也就是只需要遍历
first_name,last_name这个复合索引的索引树木就好了

using index总是和只需要索引树数据, 比如select name from t where name >'a', 或者select name from t where name = 'abbc', 就是using index, 因为只需要name并且name是可以只通过
遍历索引树木就可以了，必须要去搜全行数据(后面可能会加上using where) 


https://stackoverflow.com/questions/25672552/whats-the-difference-between-using-index-and-using-where-using-index-in-the

https://stackoverflow.com/questions/28759576/mysql-using-index-condition-vs-using-where-using-index

https://stackoverflow.com/questions/1687548/mysql-explain-using-index-vs-using-index-condition



explain一个select order by, 避免filesort, filesort表示没有排序优化

