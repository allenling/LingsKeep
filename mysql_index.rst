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

在一个有100000行的表中, 性别这一列的Cardinality为2, 则Selectivity = 2/10000 * 100% = 0.02%


3. 是否使用索引.
-------------------

数据库的query optimizers会根据Selectivity来决定是否使用该索引.

索引的Selectivity越大, query optimizers在搜索的时候有越大的几率会用到该索引.

这是因为有些情况(通常是很小的Cardinality, Selectivity和Cardinality正相关, 所以也就是很小的Selectivity)访问索引不如全表扫描, 并且访问索引也是消耗资源的, 并且有时候会更慢.

例如, 在10000列的表中, 一半是男性,一半是女性. 我们需要找到所有女性的名字, 若用性别这一个索引, 则需要访问5000次索引才能找到所有的女性, 而全表扫描的话, 每一行有50%的几率为女性.

所以, 即使某一列定义了索引, 但是在搜索中不一定会用到.

sql语句中的user index并不会一定使用指定的索引, 这个只是建议而已, 索引使用取决与query optimizers.

索引和非索引搜索区别在于线性搜索和二分搜索的效率区别.前者平局N/2, 后者log2N. 而创建索引的代价是资源的开销.

.. [#] http://www.programmerinterview.com/index.php/database-sql/selectivity-in-sql-databases/
.. [#] http://stackoverflow.com/questions/1108/how-does-database-indexing-work
.. [#] http://dev.mysql.com/doc/refman/5.5/en/mysql-indexes.html
.. [#] http://sqlmag.com/database-performance-tuning/which-faster-index-access-or-table-scan

