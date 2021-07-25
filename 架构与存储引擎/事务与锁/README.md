# MySQL 中的事务管理

InnoDB 默认的事务隔离级别是 Repeatable Read（后文中用简称 RR），它为了解决该隔离级别下的幻读的并发问题，提出了 LBCC 和 MVCC 两种方案。其中 LBCC 解决的是当前读情况下的幻读，MVCC 解决的是普通读（快照读）的幻读。

# LBCC

# 锁分类

根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。其中：

- 全局锁：对整个数据库实例加锁，常用于全库逻辑备份。
- 表锁：开销小，加锁快；不会出现死锁；锁定力度大，发生锁冲突概率高，并发度最低。
- 行锁：开销大，加锁慢；会出现死锁；锁定粒度小，发生锁冲突的概率低，并发度高；
- 页锁：开销和加锁速度介于表锁和行锁之间；会出现死锁；锁定粒度介于表锁和行锁之间，并发度一般。

每个存储引擎都可以有自己的锁策略，例如 MyISAM 引擎仅支持表级锁，而 InnoDB 引擎除了支持表级锁外，也支持行级锁（默认）。

|        | 行锁 | 表锁 | 页锁 |
| ------ | ---- | ---- | ---- |
| MyISAM |      | √    |      |
| BDB    |      | √    | √    |
| InnoDB | √    | √    |      |

MySQL 5.5 中，information_schema 库中新增了三个关于锁的表，亦即 innodb_trx、innodb_locks 和 innodb_lock_waits。其中 innodb_trx 表记录当前运行的所有事务，innodb_locks 表记录当前出现的锁，innodb_lock_waits 表记录锁等待的对应关系。

## 全局锁

全局锁即对整个数据库实例加锁。MySQL 提供加全局读锁的方法：Flush tables with read lock(FTWRL)。这个命令可以使整个库处于只读状态。使用该命令之后，数据更新语句、数据定义语句和更新类事务的提交语句等操作都会被阻塞。风险是如果在主库备份，在备份期间不能更新，业务停摆。如果在从库备份，备份期间不能执行主库同步的 binlog，导致主从延迟。官方自带的逻辑备份工具 mysqldump，当 mysqldump 使用参数--single-transaction 的时候，会启动一个事务，确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高。

## 表级锁

MySQL 里面表级锁有两种，一种是表锁，一种是元数据锁(meta data lock,MDL)。表锁的语法是: `lock tables ... read/write`，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。

另一类表级的锁是 MDL（metadata lock)。MDL 不需要显式使用，在访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。MDL 会直到事务提交才会释放，在做表结构变更的时候，一定要小心不要导致锁住线上查询和更新。

## 行锁

MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM 引擎就不支持行锁。不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB 是支持行锁的，这也是 MyISAM 被 InnoDB 替代的重要原因之一。在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。譬如事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。

# Links

- https://parg.co/LeO Mysql 事务-你想知道的都在这
- https://sctrack.sendcloud.net/track/click/eyJuZXRlYXNlIjogImZhbHNlIiwgIm1haWxsaXN0X2lkIjogMCwgInRhc2tfaWQiOiAiIiwgImVtYWlsX2lkIjogIjE2MTg1NzIxNDc3NjdfMTg3XzU3MzA0XzQ1MTYuc2MtMTBfOV8xM18yMTMtaW5ib3VuZDAkMzg0OTI0NTUyQHFxLmNvbSIsICJzaWduIjogImUxM2UxYmM4ODBiYWM0NWQyNDk3OTNkMTdkMWQzMDFlIiwgInVzZXJfaGVhZGVycyI6IHt9LCAibGFiZWwiOiAiNjE5MjAzMCIsICJ0cmFja19kb21haW4iOiAic2N0cmFjay5zZW5kY2xvdWQubmV0IiwgInJlYWxfdHlwZSI6ICIiLCAibGluayI6ICJodHRwcyUzQS8vdmlwLm1hbm9uZy5pby9ib3VuY2UlM0ZuaWQlM0Q0OSUyNmFpZCUzRDIwNTIlMjZ1cmwlM0RodHRwcyUyNTNBJTI1MkYlMjUyRnRvdXRpYW8uaW8lMjUyRmslMjUyRm9ta2I5Z3klMjZuJTNETVRNeS43MGt5ckJjZ0l1OHNJZHlnWDE4RVhRZFJFYUkiLCAib3V0X2lwIjogIjEyMC4xMzIuNTQuMTk0IiwgImNvbnRlbnRfdHlwZSI6ICIwIiwgInVzZXJfaWQiOiAxODcsICJvdmVyc2VhcyI6ICJmYWxzZSIsICJjYXRlZ29yeV9pZCI6IDYwMzQ5fQ==.html InnoDB 解决幻读的方案--LBCC&MVCC
