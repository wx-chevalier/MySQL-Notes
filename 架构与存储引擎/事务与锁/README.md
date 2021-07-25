# MySQL 中的事务管理

InnoDB 默认的事务隔离级别是 Repeatable Read（后文中用简称 RR），它为了解决该隔离级别下的幻读的并发问题，提出了 LBCC 和 MVCC 两种方案。

- InnoDB 中使用索引作为检索条件修改数据时采用行锁，否则使用表锁
- InnoDB 自动给修改操作加锁，给查询操作不自动加锁
- 在 REPEATABLE READ 级别下，如果要完全杜绝幻读，需要手动给关键查询语句加锁；LBCC 解决的是当前读情况下的幻读，MVCC 解决的是普通读（快照读）的幻读
- 表的大部分数据需要修改时，行锁反而不如表锁更有效率

# LBCC

在 MySQL 中，为了应对并发场景下的读写，锁通常分为两类：共享锁以及排他锁。其中，共享锁允许多个连接在同一时间并发的读取相同的资源，彼此之间互不影响,所以又称为读锁。排他锁则会阻塞其他尝试获取共享锁或者排他锁的操作，确保同一时间只有一个连接可以写入数据，并禁止其他用户的读写，又称写锁。

在实际使用下，加锁往往意味着高昂的开销，MySQL 为了平衡锁的开销以及并发的线程之间的安全，采用了两种不同的锁策略：

- 表锁：表锁会锁定整张表，如果当前有用户正在执行写操作并且获取了写锁，这可能导致整张表被锁定，阻塞其他用户的读写操作。如果用户执行的是读操作，则会获取读锁，此时其他用户的并发读操作将被接受，写操作会被阻塞。以 `UPDATE table SET a = 1 where b = 2;` 为例，如果 b 字段不存在索引，那么会锁住所有的记录，即锁上了表锁。
- 行锁：行锁的粒度是在每一条行数据，这意味行锁可以尽可能的支持并发处理，相应的行锁开销也会比较大。并且，在 InnoDB 中的行锁是针对索引加的锁，不是针对记录加的锁，并且该索引不能失效，否则行锁将会自动升级为表锁。

相比较而言，表锁的优势在于开销小，加锁快，无死锁，劣势是锁的粒度大，发生锁冲突的概率较高，并发能力较弱。而行锁则相反。实际使用中，两者都会由 MySQL 自动加锁。行锁冲突可以通过执行 show status like 'innodb_row_lock%'语句进行分析，表锁冲突则可通过执行 show status like 'table_locks%' 进行查看。

# MVCC

MVCC(multiple-version-concurrency-control）是个行级锁的变种，它在普通读情况下避免了加锁操作，因此开销更低。其原理具体为，在 InnoDB 存储引擎中，每行数据会加入一些隐藏字段 DATA_TRX_ID，DATA_ROLL_PTR，DB_ROW_ID，DELETE_BIT。DATA_TRX_ID 字段记录了数据的创建和删除时间，这个时间指的是对数据进行操作的事务的 id，DATA_ROLL_PTR 指向当前数据的 undo log 记录，回滚数据就是通过这个指针，DELETE BIT 位用于标识该记录是否被删除，这里的不是真正的删除数据，而是标志出来的删除。真正意义的删除是在 MySQL 进行数据的 GC，清理历史版本数据的时候。

相应的，其 DML 的处理方式也发生了变化：

- SELECT 语句先查找 DATA_TRX_ID 早于当前事务 ID 的数据行。这样就保证了读取的数据要么是在这个事务开始之前就已经 commit 了的（早于当前事务 ID），要么是在这个事务中自身创建的数据（等于当前事务 ID）。查找行的 DELETE_BIT 为 1 时，查找删除事务 ID 对应的事务，确定此条记录在当前事务开始之前，行没有被删除。
- INSERT 语句会在新插入行数据之后，保存当前事务 ID 作为行的 DATA_TRX_ID。
- DELETE 语句为每一条删除的记录保存当前的事务 ID 作为行的删除标记。
- UPDATE 语句将复制变更的记录，并把新记录的 DATA_TRX_ID 置为当前事务 ID，同时更新老记录中的 DB_ROLL_PT 指向了上一个版本。

所以在并发读的时候，不需要等到访问行上的锁释放，只需要读取一个行的快照即可。既然是多版本的读取，就肯定读取不到其他事务中的新插入的数据了，也就避免了上述场景中提到的幻读。避免了部分幻读现象，但是实际使用中，还是会有幻读产生，先看场景：

```sql
-- 会话 1
START TRANSACTION;

SELECT * FROM xx;
-- 此时查询表为空，且事务未提交

-- 会话 2
START TRANSACTION;

SELECT * FROM xx;
-- 此时查询表为空，且事务未提交

INSERT INTO t_bitfly VALUES (1, 'test');
-- 插入一条主键为 1 的记录

commit;
-- 提交会话 2 中的事务

-- 会话 1
INSERT INTO t_bitfy VALUES(1, 'test');
-- 尝试插入主键为 1 的一条记录，此时会受到主键重复的报错，但是再查询语句中明明没有这条记录，幻读出现

```

通过 MVCC，在事务中的多次读取不会出现幻读，但是此时的插入操作依旧会发生主键重复的错误，并且因为 MVCC 机制，在上图中的会话 1 无论读取多少次都不会读到导致冲突产生的数据，确实就如“幻影”一般诡异。为了解决上述场景中的幻读，需要简单提一下 InnoDB 的行锁机制，在 InnoDB 引擎下存在三种行锁，分别为：

- Record Lock：在单行记录上的锁
- Gap Lock：间隙锁，锁定一个范围，但不包括记录本身，。GAP 锁的目的，是为了防止同一事务的两次读出现幻读的情况
- Next-Key Lock: 前两个锁的共同使用，即锁定了记录本身，也锁定了一定的范围。

通常情况下，INSERT/UPDATE/DELETE 默认会在操作的记录上加上 Next-Key Lock，而普通的 SELECT 因为 MVCC 的关系反而只需要读取快照即可，所以如果业务需要再 REPEATABLE READ 场景下保证绝对不产生幻读，需要手动给 SELECT 加锁，在类似 SELECT…WHERE 加入 FOR UPDATE（排它锁）或者 LOCK IN SHARE MODE（共享锁）。

# Links

- https://parg.co/LeO Mysql 事务-你想知道的都在这
- https://sctrack.sendcloud.net/track/click/eyJuZXRlYXNlIjogImZhbHNlIiwgIm1haWxsaXN0X2lkIjogMCwgInRhc2tfaWQiOiAiIiwgImVtYWlsX2lkIjogIjE2MTg1NzIxNDc3NjdfMTg3XzU3MzA0XzQ1MTYuc2MtMTBfOV8xM18yMTMtaW5ib3VuZDAkMzg0OTI0NTUyQHFxLmNvbSIsICJzaWduIjogImUxM2UxYmM4ODBiYWM0NWQyNDk3OTNkMTdkMWQzMDFlIiwgInVzZXJfaGVhZGVycyI6IHt9LCAibGFiZWwiOiAiNjE5MjAzMCIsICJ0cmFja19kb21haW4iOiAic2N0cmFjay5zZW5kY2xvdWQubmV0IiwgInJlYWxfdHlwZSI6ICIiLCAibGluayI6ICJodHRwcyUzQS8vdmlwLm1hbm9uZy5pby9ib3VuY2UlM0ZuaWQlM0Q0OSUyNmFpZCUzRDIwNTIlMjZ1cmwlM0RodHRwcyUyNTNBJTI1MkYlMjUyRnRvdXRpYW8uaW8lMjUyRmslMjUyRm9ta2I5Z3klMjZuJTNETVRNeS43MGt5ckJjZ0l1OHNJZHlnWDE4RVhRZFJFYUkiLCAib3V0X2lwIjogIjEyMC4xMzIuNTQuMTk0IiwgImNvbnRlbnRfdHlwZSI6ICIwIiwgInVzZXJfaWQiOiAxODcsICJvdmVyc2VhcyI6ICJmYWxzZSIsICJjYXRlZ29yeV9pZCI6IDYwMzQ5fQ==.html InnoDB 解决幻读的方案--LBCC&MVCC
