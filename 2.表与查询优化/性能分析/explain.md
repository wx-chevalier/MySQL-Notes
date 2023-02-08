# Explain

MySQL 提供了一个 EXPLAIN 命令, 它可以对 SELECT 语句进行分析, 并输出 SELECT 执行的详细信息, 以供开发人员针对性优化.EXPLAIN 命令用法十分简单, 在 SELECT 语句前加上 explain 就可以了, 例如:

```sql
explain SELECT id FROM ORDER_EXPENSE_SUMMARY
WHERE STATUS = 'CHECKED' AND EXPENSE_BILL_NO = 'BF_20191112_20191118_PAY_1'
ORDER BY GMT_CREATE DESC
```

- id :表示 SQL 执行的顺序的标识,SQL 从大到小的执行。示例中 1 表示执行的第一条 SQL

- select_type :表示 select 语句的子类型。

- type :表示访问类型，显示出该查询是通过全表扫描、索引查询等方式查找数据的。

- possible_keys :显示查询可能使用了哪些索引，表示该索引可以进行高效地查找，但是列出来的索引对于后续 优化过程可能是没有用的。

- key : 表示 MySQL 实际决定使用的键（索引）。如果没有选择索引，键是 NULL。

- key_len : 表示 key_len 列显示 MySQL 决定使用的键长度。使用的索引的长度。在不损失精确性的情况下，长度越短越好。

- ref :表示使用哪个列或查询参数 被用来做索引的查询条件

- rows :表示 MySQL 认为它执行查询时必须检查的行数。这是一个预估值，值越少说明查询效率越高。

- Extra :表示 MySQL 查询优化器执行查询的过程中对查询计划的重要补充信息。

其中 type、key、rows、Extra 是需要重点关注的字段。例如 type 为 ALL 表示全表扫描，那考虑是不是需要增加索引。key 为 null 那么是不是索引缺失或者没有命中，Extra 里有 Using filesort，那么说明查询的原始结果会在内存里面做排序，效率比较低。

# 索引分析

# 索引查询

联合索引中，索引字段的数据必须是有序的，才能实现这种类型的查找，才能利用到索引。

```sql
--- UNIQUE KEY `unique_product_in_category` (`name`,`category`) USING BTREE,
--- key 为 unique_name_in_category
EXPLAIN SELECT * FROM product WHERE name = '产品一'；
--- key 为 null
EXPLAIN SELECT * FROM product WHERE category = '类目一'；
```

- SYSTEM，CONST 的特例，当表上只有一条元组匹配

- CONST，WHERE 条件筛选后表上至多有一条元组匹配时，比如 WHERE ID = 2（ID 是主键，值为 2 的要么有一条要么没有）

- EQ_REF，参与连接运算的表是内表（在代码实现的算法中，两表连接时作为循环中的内循环遍历的对象，这样的表称为内表）。

基于索引（连接字段上存在唯一索引或者主键索引，且操作符必须是“=”谓词，索引值不能为 NULL）做扫描，使得对外表的一条元组，内表只有唯一一条元组与之对应。

（4）REF
可以用于单表扫描或者连接。参与连接运算的表，是内表。

基于索引（连接字段上的索引是非唯一索引，操作符必须是“=”谓词，连接字段值不可为 NULL）做扫描，使得对外表的一条元组，内表可有若干条元组与之对应。

（5）REF_OR_NULL
类似 REF，只是搜索条件包括：连接字段的值可以为 NULL 的情况，比如 where col = 2 or col is null

（6）RANGE
范围扫描，基于索引做范围扫描，为诸如 BETWEEN，IN，>=，LIKE 类操作提供支持

（7）INDEX_SCAN
索引做扫描，是基于索引在索引的叶子节点上找满足条件的数据（不需要访问数据文件）

（8）ALL
全表扫描或者范围扫描：不使用索引，顺序扫描，直接读取表上的数据（访问数据文件）

（9）UNIQUE_SUBQUERY
在子查询中，基于唯一索引进行扫描，类似于 EQ_REF

（10）INDEX_SUBQUERY
在子查询中，基于除唯一索引之外的索引进行扫描

（11）INDEX_MERGE
多重范围扫描。两表连接的每个表的连接字段上均有索引存在且索引有序，结果合并在一起。适用于作集合的并、交操作。

（12）FT
FULL TEXT，全文检索
