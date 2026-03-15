# MySQL 性能分析

MySQL 数 据库是常见的两个瓶颈是 CPU 和 IO 的瓶颈，CPU 在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候。磁盘 IO 瓶颈发生在装入数据远大于内 存容量的时候，如果应用分布在网络上，那么查询量相当大的时候那么平瓶颈就会出现在网络上，我们可以用 mpstat, iostat, sar 和 vmstat 来查看系统的性能状态。除了服务器硬件的性能瓶颈，对于 MySQL 系统本身，我们可以使用工具来优化数据库的性能，通常有三种：使用索引，使用 EXPLAIN 分析查询以及调整 MySQL 的内部配置。

![](http://img.blog.csdn.net/20160518221004236)

# 性能瓶颈定位

## 存储占用分析

```sql
-- 统计表、索引 大小
SELECT ISS.SCHEMA_NAME AS "Schema_Name",

       ITS.TABLE_NAME AS "Table_Name",

       (ITS.DATA_LENGTH / 1024 / 1024) AS "Data(MB)",

       (ITS.INDEX_LENGTH / 1024 / 1024) AS "Index(MB)",

       ((ITS.DATA_LENGTH + ITS.INDEX_LENGTH) / 1024 / 1024) AS "Data+Index(MB)",

       ITS.TABLE_ROWS AS "Total_Rows"

  FROM `information_schema`.`TABLES` ITS RIGHT JOIN

       `information_schema`.`SCHEMATA` ISS

    ON ITS.TABLE_SCHEMA = ISS.SCHEMA_NAME

 WHERE ISS.SCHEMA_NAME LIKE "x%"

 ORDER BY 4 DESC, ISS.SCHEMA_NAME, ITS.TABLE_NAME;
```

## Show

> - [MySQL 性能查看(命中率，慢查询)](http://blog.csdn.net/iquicksandi/article/details/7970706)

我们可以通过 show 命令查看 MySQL 状态及变量，找到系统的瓶颈：

```
Mysql> show status ——显示状态信息(扩展show status like 'XXX')
Mysql> show variables ——显示系统变量(扩展show variables like 'XXX')
Mysql> show innodb status ——显示InnoDB存储引擎的状态
Mysql> show processlist ——查看当前SQL执行，包括执行状态、是否锁表等
Shell> mysqladmin variables -u username -p password——显示系统变量
Shell> mysqladmin extended-status -u username -p password——显示状态信息
```

查看状态变量及帮助：

```
Shell> mysqld --verbose --help [|more #逐行显示]
```

## 慢查询日志

### 日志开启

在配置文件 my.cnf 或 my.ini 中在[mysqld]一行下面加入两个配置参数

```
log-slow-queries=/data/mysqldata/slow-query.log
long_query_time=2
```

log-slow-queries 参数为慢查询日志存放的位置，一般这个目录要有 mysql 的运行帐号的可写权限，一般都将这个目录设置为 mysql 的数据存放目录；long_query_time=2 中的 2 表示查询超过两秒才记录；在 my.cnf 或者 my.ini 中添加 log-queries-not-using-indexes 参数，表示记录下没有使用索引的查询。

```
log-slow-queries=/data/mysqldata/slow-query.log
long_query_time=10
log-queries-not-using-indexes
```

我们可以通过命令行设置变量来即时启动慢日志查询。由下图可知慢日志没有打开，slow_launch_time=# 表示如果建立线程花费了比这个值更长的时间,slow_launch_threads 计数器将增加：

![](http://www.2cto.com/uploadfile/2011/1020/20111020040037661.jpg)

设置慢日志开启:

![](http://www.2cto.com/uploadfile/2011/1020/20111020040037956.jpg)

MySQL 后可以查询 long_query_time 的值。

![](http://www.2cto.com/uploadfile/2011/1020/20111020040037528.jpg)

为了方便测试，可以将修改慢查询时间为 5 秒。

![](http://www.2cto.com/uploadfile/2011/1020/20111020040037987.jpg)

### 日志分析

我们可以通过打开 log 文件查看得知哪些 SQL 执行效率低下

```
[root@localhost mysql]# more slow-query.log
# Time: 081026 19:46:34
# User@Host: root[root] @ localhost []
# Query_time: 11 Lock_time: 0 Rows_sent: 1 Rows_examined: 6552961
select count(*) from t_user;
```

从日志中，可以发现查询时间超过 5 秒的 SQL，而小于 5 秒的没有出现在此日志中。如果慢查询日志中记录内容很多，可以使用 mysqldumpslow 工具(MySQL 客户端安装自带)来对慢查询日志进行分类汇总。mysqldumpslow 对日志文件进行了分类汇总，显示汇总后摘要结果。进入 log 的存放目录，运行

```
[root@mysql_data]#mysqldumpslow  slow-query.log
Reading mysql slow query log from slow-query.log
Count: 2 Time=11.00s (22s) Lock=0.00s (0s) Rows=1.0 (2), root[root]@mysql
select count(N) from t_user;
```

mysqldumpslow 命令

```
/path/mysqldumpslow -s c -t 10 /database/mysql/slow-query.log
```

这会输出记录次数最多的 10 条 SQL 语句，其中：

- -s, 是表示按照何种方式排序，c、t、l、r 分别是按照记录次数、时间、查询时间、返回的记录数来排序，ac、at、al、ar，表示相应的倒叙；
- -t, 是 top n 的意思，即为返回前面多少条的数据；
- -g, 后边可以写一个正则匹配模式，大小写不敏感的；
  例如：

```
/path/mysqldumpslow -s r -t 10 /database/mysql/slow-log
```

得到返回记录集最多的 10 个查询。

```
/path/mysqldumpslow -s t -t 10 -g “left join” /database/mysql/slow-log
```

得到按照时间排序的前 10 条里面含有左连接的查询语句。使用 mysqldumpslow 命令可以非常明确的得到各种我们需要的查询语句，对 MySQL 查询语句的监控、分析、优化是 MySQL 优化非常重要的一 步。开启慢查询日志后，由于日志记录操作，在一定程度上会占用 CPU 资源影响 mysql 的性能，但是可以阶段性开启来定位性能瓶颈。
