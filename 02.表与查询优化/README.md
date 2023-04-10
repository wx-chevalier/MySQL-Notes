# MySQL 性能优化

MySQL 的优化主要分为表结构优化（Scheme optimization）和查询优化（Query optimization），在结构优化与查询优化中都需要进行索引优化。

# 慢 SQL 分析原则

治理慢 SQL 的方法，主要分三类，一类是纯粹基于 SQL 语句的结构治理，第二类是结合业务场景治理慢 SQL，第三类是业务或者结构的优化：

- 查询的条件区分度高不高。区分度的公式是 `count(distinct col)/count(*)`，表示字段不重复的比例，比例越大扫描的记录数越少，所以尽选择区分度高的列作为索引,唯一键的区分度是 1，而一些状态、性别字段可能在大数据面前区分度就是 0。一般对于区分度大于 0.1 的查询字段都要建立索引。
- 数据本身的倾斜度大不大，虽然字段的区分度有时候并不能反映数据的均匀情况，如果数据非常不均衡，也可能会导致慢 SQL，例如用户刷单，这个用户下的单量非常大，那么基于 userId 去查询用户订单的执行速度也会很慢。

## 索引失效

- 最左前缀匹配原则。mysql 会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如 a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d 是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d 的顺序可以任意调整；需要范围查询的字段放在索引的尾巴上。
- 索引列不能参与计算，保持列“干净”。比如 `from_unixtime(create_time) = ’2019-12-01’` 就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成 `create_time = unix_timestamp(’2019-12-01’);`
- 使用查询语句 where 条件时，不允许出现函数，否则索引会失效。
- LIKE 语句不允许使用 % 开头，否则索引会失效；
- 组合索引一定要遵循 从左到右 原则，否则索引会失效；比如：`SELECT * FROM table WHERE name = '张三' AND age > 18`，那么该组合索引必须是 name,age 形式；
- 索引字段的数据类型和查询的数据类型已经要匹配上，不然索引也会失效。
- 尽量扩展索引、不要新建索引。

## 业务或架构的优化

有一些情况的确通过索引或者修改 SQL 语句无法解决问题，比如常见的一些 case。

- 由于单表的数据量过大，例如达到千万级别的数据了，可能需要分库分表才能解决。
- 有 like 的全模糊的查询，比如基于文本内容去查订单信息，需要接 openSearch 的解决。
- 有一些统计查询类的后台功能，考虑是不是要单独维护一张统计表。
- 有热点数据的查询，考虑是否要接 Tair 等缓存解决。
- 有些场景是不是 Mysql 不适用，需要用 K-V 的数据库，HBASE 等非结构化的存储引擎。

# Links

- https://www.itcodemonkey.com/article/14204.html

- https://www.itcodemonkey.com/article/13753.html

- https://mp.weixin.qq.com/s/ZGgoFq2zvkOMXAE_WWYvQA

- https://www.imooc.com/learn/194

- https://www.itcodemonkey.com/article/13851.html

- http://blog.codinglabs.org/articles/theory-of-mysql-index.html 利用 test_db 分析索引的使用情况系列

- https://mp.weixin.qq.com/s/Edn_gPwcAHo5sYIzLJghzA

- https://mp.weixin.qq.com/s/ajYl2eBrgqc9cBd9_OH3_w?from=groupmessage&isappinstalled=0

- https://mp.weixin.qq.com/s/AYnP-Sp82yYguCn0Gf8cIg

- https://mp.weixin.qq.com/s/Qv5QzzVoUtIB58UmIRG-lQ

- https://zhuanlan.zhihu.com/p/72071609

- https://mp.weixin.qq.com/s/UkQ31grdLZJttlNAPFs9cg

- https://mp.weixin.qq.com/s/3yki4dljbLMgnOVrsqbk8w 慢 SQL 排查思路？就这。
