# 数据查询

完整的 SQL 查询语句语法如下：

```sql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_condition]
    [WINDOW window_name AS (window_spec)
        [, window_name AS (window_spec)] ...]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR {UPDATE | SHARE} [OF tbl_name [, tbl_name] ...] [NOWAIT | SKIP LOCKED]
      | LOCK IN SHARE MODE]]
```

# SQL 查询流程

![SQL 查询流程](https://assets.ng-tech.icu/item/20230622202543.png)

SQL 语句由数据库系统分几个步骤执行，包括：

- 解析 SQL 语句并检查其有效性
- 将 SQL 转换为内部表示，如关系代数
- 优化内部表示并创建一个利用索引信息的执行计划
- 执行该计划并返回结果

SQL 的执行是非常复杂的，涉及到许多考虑因素，例如：

- 索引和缓存的使用
- 表连接的顺序
- 并发控制
- 事务管理
