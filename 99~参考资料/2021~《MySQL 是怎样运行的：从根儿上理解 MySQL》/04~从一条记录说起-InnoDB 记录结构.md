# 第 4 章 从一条记录说起-InnoDB 记录结构

## 准备工作

&emsp;&emsp;到现在为止，`MySQL`对于我们来说还是一个黑盒，我们只负责使用客户端发送请求并等待服务器返回结果，表中的数据到底存到了哪里？以什么格式存放的？`MySQL`是以什么方式来访问的这些数据？这些问题我们统统不知道，对于未知领域的探索向来就是社会主义核心价值观中的一部分，作为新一代社会主义接班人，不把它们搞懂怎么支援祖国建设呢？

&emsp;&emsp;我们前面介绍请求处理过程的时候提到过，`MySQL`服务器上负责对表中数据的读取和写入工作的部分是`存储引擎`，而服务器又支持不同类型的存储引擎，比如`InnoDB`、`MyISAM`、`Memory`什么的，不同的存储引擎一般是由不同的人为实现不同的特性而开发的，<span style="color:red">真实数据在不同存储引擎中存放的格式一般是不同的</span>，甚至有的存储引擎比如`Memory`都不用磁盘来存储数据，也就是说关闭服务器后表中的数据就消失了。由于`InnoDB`是`MySQL`默认的存储引擎，也是我们最常用到的存储引擎，我们也没有那么多时间去把各个存储引擎的内部实现都看一遍，所以本集要介绍的是使用`InnoDB`作为存储引擎的数据存储结构，了解了一个存储引擎的数据存储结构之后，其他的存储引擎都是依葫芦画瓢，等我们用到了再说。

## InnoDB 页简介

&emsp;&emsp;`InnoDB`是一个将表中的数据存储到磁盘上的存储引擎，所以即使关机后重启我们的数据还是存在的。而真正处理数据的过程是发生在内存中的，所以需要把磁盘中的数据加载到内存中，如果是处理写入或修改请求的话，还需要把内存中的内容刷新到磁盘上。而我们知道读写磁盘的速度非常慢，和内存读写差了几个数量级，所以当我们想从表中获取某些记录时，`InnoDB`存储引擎需要一条一条的把记录从磁盘上读出来么？不，那样会慢死，`InnoDB`采取的方式是：<span style="color:red">将数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位，InnoDB 中页的大小一般为 **_16_** KB</span>。也就是在一般情况下，一次最少从磁盘中读取 16KB 的内容到内存中，一次最少把内存中的 16KB 内容刷新到磁盘中。

## InnoDB 行格式

&emsp;&emsp;我们平时是以记录为单位来向表中插入数据的，这些记录在磁盘上的存放方式也被称为`行格式`或者`记录格式`。设计`InnoDB`存储引擎的大佬们到现在为止设计了 4 种不同类型的`行格式`，分别是`Compact`、`Redundant`、`Dynamic`和`Compressed`行格式，随着时间的推移，他们可能会设计出更多的行格式，但是不管怎么变，在原理上大体都是相同的。

### 指定行格式的语法

&emsp;&emsp;我们可以在创建或修改表的语句中指定`行格式`：

```
CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称

ALTER TABLE 表名 ROW_FORMAT=行格式名称
```

&emsp;&emsp;比如我们在`xiaohaizi`数据库里创建一个演示用的表`record_format_demo`，可以这样指定它的`行格式`：

```
mysql> USE xiaohaizi;
Database changed

mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
Query OK, 0 rows affected (0.03 sec)
```

&emsp;&emsp;可以看到我们刚刚创建的这个表的`行格式`就是`Compact`，另外，我们还显式指定了这个表的字符集为`ascii`，因为`ascii`字符集只包括空格、标点符号、数字、大小写字母和一些不可见字符，所以我们的汉字是不能存到这个表里的。我们现在向这个表中插入两条记录：

```
mysql> INSERT INTO record_format_demo(c1, c2, c3, c4) VALUES('aaaa', 'bbb', 'cc', 'd'), ('eeee', 'fff', NULL, NULL);
Query OK, 2 rows affected (0.02 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

&emsp;&emsp;现在表中的记录就是这个样子的：

```
mysql> SELECT * FROM record_format_demo;
+------+-----+------+------+
| c1   | c2  | c3   | c4   |
+------+-----+------+------+
| aaaa | bbb | cc   | d    |
| eeee | fff | NULL | NULL |
+------+-----+------+------+
2 rows in set (0.00 sec)

mysql>
```

&emsp;&emsp;演示表的内容也填充好了，现在我们就来看看各个行格式下的存储方式到底有什么不同吧～

### COMPACT 行格式

&emsp;&emsp;废话不多说，直接看图：

![COMPACT行格式][04-01]

&emsp;&emsp;大家从图中可以看出来，一条完整的记录其实可以被分为`记录的额外信息`和`记录的真实数据`两大部分，下面我们详细看一下这两部分的组成。

#### 记录的额外信息

&emsp;&emsp;这部分信息是<span style="color:red">服务器为了描述这条记录而不得不额外添加的一些信息</span>，这些额外信息分为 3 类，分别是`变长字段长度列表`、`NULL值列表`和`记录头信息`，我们分别看一下。

##### 变长字段长度列表

&emsp;&emsp;我们知道`MySQL`支持一些变长的数据类型，比如`VARCHAR(M)`、`VARBINARY(M)`、各种`TEXT`类型，各种`BLOB`类型，我们也可以把拥有这些数据类型的列称为`变长字段`，变长字段中存储多少字节的数据是不固定的，所以我们在存储真实数据的时候需要顺便把这些数据占用的字节数也存起来，这样才不至于把`MySQL`服务器搞懵，所以这些变长字段占用的存储空间分为两部分：

1. 真正的数据内容
2. 占用的字节数

&emsp;&emsp;在`Compact`行格式中，<span style="color:red">把所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表，各变长字段数据占用的字节数按照列的顺序逆序存放</span>，我们再次强调一遍，是<span style="color:red">逆序</span>存放！

&emsp;&emsp;我们拿`record_format_demo`表中的第一条记录来举个例子。因为`record_format_demo`表的`c1`、`c2`、`c4`列都是`VARCHAR(10)`类型的，也就是变长的数据类型，所以这三个列的值的长度都需要保存在记录开头处，因为`record_format_demo`表中的各个列都使用的是`ascii`字符集，所以每个字符只需要 1 个字节来进行编码，来看一下第一条记录各变长字段内容的长度：

| 列名 | 存储内容 | 内容长度（十进制表示） | 内容长度（十六进制表示） |
| :--: | :------: | :--------------------: | :----------------------: |
| `c1` | `'aaaa'` |          `4`           |          `0x04`          |
| `c2` | `'bbb'`  |          `3`           |          `0x03`          |
| `c4` |  `'d'`   |          `1`           |          `0x01`          |

&emsp;&emsp;又因为这些长度值需要按照列的<span style="color:red">逆序</span>存放，所以最后`变长字段长度列表`的字节串用十六进制表示的效果就是（各个字节之间实际上没有空格，用空格隔开只是方便理解）：

```
01 03 04
```

&emsp;&emsp;把这个字节串组成的`变长字段长度列表`填入上面的示意图中的效果就是：

![填入变长字段长度列表][04-02]

&emsp;&emsp;由于第一行记录中`c1`、`c2`、`c4`列中的字符串都比较短，也就是说内容占用的字节数比较小，用 1 个字节就可以表示，但是如果变长列的内容占用的字节数比较多，可能就需要用 2 个字节来表示。具体用 1 个还是 2 个字节来表示真实数据占用的字节数，`InnoDB`有它的一套规则，我们首先声明一下`W`、`M`和`L`的意思：

1. 假设某个字符集中表示一个字符最多需要使用的字节数为`W`，也就是使用`SHOW CHARSET`语句的结果中的`Maxlen`列，比方说`utf8`字符集中的`W`就是`3`，`gbk`字符集中的`W`就是`2`，`ascii`字符集中的`W`就是`1`。

2. 对于变长类型`VARCHAR(M)`来说，这种类型表示能存储最多`M`个字符（注意是字符不是字节），所以这个类型能表示的字符串最多占用的字节数就是`M×W`。
3. 假设它实际存储的字符串占用的字节数是`L`。

&emsp;&emsp;所以确定使用 1 个字节还是 2 个字节表示真正字符串占用的字节数的规则就是这样：

- 如果`M×W <= 255`，那么使用 1 个字节来表示真正字符串占用的字节数。

```
    就是说InnoDB在读记录的变长字段长度列表时先查看表结构，如果某个变长字段允许存储的最大字节数不大于255时，可以认为只使用1个字节来表示真正字符串占用的字节数。
```

- 如果`M×W > 255`，则分为两种情况：
  - 如果`L <= 127`，则用 1 个字节来表示真正字符串占用的字节数。
  - 如果`L > 127`，则用 2 个字节来表示真正字符串占用的字节数。

```
    InnoDB在读记录的变长字段长度列表时先查看表结构，如果某个变长字段允许存储的最大字节数大于255时，该怎么区分它正在读的某个字节是一个单独的字段长度还是半个字段长度呢？设计InnoDB的大佬使用该字节的第一个二进制位作为标志位：如果该字节的第一个位为0，那该字节就是一个单独的字段长度（使用一个字节表示不大于127的二进制的第一个位都为0），如果该字节的第一个位为1，那该字节就是半个字段长度。
    对于一些占用字节数非常多的字段，比方说某个字段长度大于了16KB，那么如果该记录在单个页面中无法存储时，InnoDB会把一部分数据存放到所谓的溢出页中（我们后边会介绍），在变长字段长度列表处只存储留在本页面中的长度，所以使用两个字节也可以存放下来。
```

&emsp;&emsp;总结一下就是说：如果该可变字段允许存储的最大字节数（`M×W`）超过 255 字节并且真实存储的字节数（`L`）超过 127 字节，则使用 2 个字节，否则使用 1 个字节。

&emsp;&emsp;另外需要注意的一点是，<span style="color:red">变长字段长度列表中只存储值为 **_非 NULL_** 的列内容占用的长度，值为 **_NULL_** 的列的长度是不储存的 </span>。也就是说对于第二条记录来说，因为`c4`列的值为`NULL`，所以第二条记录的`变长字段长度列表`只需要存储`c1`和`c2`列的长度即可。其中`c1`列存储的值为`'eeee'`，占用的字节数为`4`，`c2`列存储的值为`'fff'`，占用的字节数为`3`。数字`4`可以用 1 个字节表示，`3`也可以用 1 个字节表示，所以整个`变长字段长度列表`共需 2 个字节。填充完`变长字段长度列表`的两条记录的对比图如下：

![填充完`变长字段长度列表`的两条记录的对比图][04-03]

```
小贴士：并不是所有记录都有这个 变长字段长度列表 部分，比方说表中所有的列都不是变长的数据类型的话，这一部分就不需要有。
```

##### NULL 值列表

&emsp;&emsp;我们知道表中的某些列可能存储`NULL`值，如果把这些`NULL`值都放到`记录的真实数据`中存储会很占地方，所以`Compact`行格式把这些值为`NULL`的列统一管理起来，存储到`NULL`值列表中，它的处理过程是这样的：

1. 首先统计表中允许存储`NULL`的列有哪些。

   &emsp;&emsp;我们前面说过，主键列、被`NOT NULL`修饰的列都是不可以存储`NULL`值的，所以在统计的时候不会把这些列算进去。比方说表`record_format_demo`的 3 个列`c1`、`c3`、`c4`都是允许存储`NULL`值的，而`c2`列是被`NOT NULL`修饰，不允许存储`NULL`值。

2. <span style="color:red">如果表中没有允许存储 **_NULL_** 的列，则 _NULL 值列表_ 也不存在了</span>，否则将每个允许存储`NULL`的列对应一个二进制位，二进制位按照列的顺序<span style="color:red">逆序</span>排列，二进制位表示的意义如下：

   - 二进制位的值为`1`时，代表该列的值为`NULL`。
   - 二进制位的值为`0`时，代表该列的值不为`NULL`。

   &emsp;&emsp;因为表`record_format_demo`有 3 个值允许为`NULL`的列，所以这 3 个列和二进制位的对应关系就是这样：

   ![record_format_demo表3值对应关系][04-04]

   &emsp;&emsp;再一次强调，二进制位按照列的顺序<span style="color:red">逆序</span>排列，所以第一个列`c1`和最后一个二进制位对应。

3. `MySQL`规定`NULL值列表`必须用整数个字节的位表示，如果使用的二进制位个数不是整数个字节，则在字节的高位补`0`。

   &emsp;&emsp;表`record_format_demo`只有 3 个值允许为`NULL`的列，对应 3 个二进制位，不足一个字节，所以在字节的高位补`0`，效果就是这样：

   ![][04-05]

   &emsp;&emsp;以此类推，如果一个表中有 9 个允许为`NULL`，那这个记录的`NULL`值列表部分就需要 2 个字节来表示了。

&emsp;&emsp;知道了规则之后，我们再返回头看表`record_format_demo`中的两条记录中的`NULL值列表`应该怎么储存。因为只有`c1`、`c3`、`c4`这 3 个列允许存储`NULL`值，所以所有记录的`NULL值列表`只需要一个字节。

- 对于第一条记录来说，`c1`、`c3`、`c4`这 3 个列的值都不为`NULL`，所以它们对应的二进制位都是`0`，画个图就是这样：

  ![][04-06]
  &emsp;&emsp;所以第一条记录的`NULL值列表`用十六进制表示就是：`0x00`。

- 对于第二条记录来说，`c1`、`c3`、`c4`这 3 个列中`c3`和`c4`的值都为`NULL`，所以这 3 个列对应的二进制位的情况就是：

  ![][04-07]

  &emsp;&emsp;所以第二条记录的`NULL值列表`用十六进制表示就是：`0x06`。

&emsp;&emsp;所以这两条记录在填充了`NULL值列表`后的示意图就是这样：

![][04-08]

##### 记录头信息

&emsp;&emsp;除了`变长字段长度列表`、`NULL值列表`之外，还有一个用于描述记录的`记录头信息`，它是由固定的`5`个字节组成。`5`个字节也就是`40`个二进制位，不同的位代表不同的意思，如图：

![][04-09]

&emsp;&emsp;这些二进制位代表的详细信息如下表：

|      名称      | 大小（单位：bit） |                                               描述                                                |
| :------------: | :---------------: | :-----------------------------------------------------------------------------------------------: |
|   `预留位1`    |        `1`        |                                             没有使用                                              |
|   `预留位2`    |        `1`        |                                             没有使用                                              |
| `delete_mask`  |        `1`        |                                       标记该记录是否被删除                                        |
| `min_rec_mask` |        `1`        |                          B+树的每层非叶子节点中的最小记录都会添加该标记                           |
|   `n_owned`    |        `4`        |                                     表示当前记录拥有的记录数                                      |
|   `heap_no`    |       `13`        |                                  表示当前记录在记录堆的位置信息                                   |
| `record_type`  |        `3`        | 表示当前记录的类型，`0`表示普通记录，`1`表示 B+树非叶子节点记录，`2`表示最小记录，`3`表示最大记录 |
| `next_record`  |       `16`        |                                     表示下一条记录的相对位置                                      |

&emsp;&emsp;大家不要被这么多的属性和陌生的概念给吓着，我这里只是为了内容的完整性把这些位代表的意思都写了出来，现在没必要把它们的意思都记住，记住也没什么用，现在只需要看一遍混个脸熟，等之后用到这些属性的时候我们再回过头来看。

&emsp;&emsp;因为我们并不清楚这些属性详细的用法，所以这里就不分析各个属性值是怎么产生的了，之后我们遇到会详细看的。所以我们现在直接看一下`record_format_demo`中的两条记录的`头信息`分别是什么：

![][04-10]

```
小贴士：再一次强调，大家如果看不懂记录头信息里各个位代表的概念千万别纠结，我们后边会说的～
```

#### 记录的真实数据

&emsp;&emsp;对于`record_format_demo`表来说，`记录的真实数据`除了`c1`、`c2`、`c3`、`c4`这几个我们自己定义的列的数据以外，`MySQL`会为每个记录默认的添加一些列（也称为`隐藏列`），具体的列如下：

|       列名       | 是否必须 | 占用空间 |          描述           |
| :--------------: | :------: | :------: | :---------------------: |
|     `row_id`     |    否    | `6`字节  | 行 ID，唯一标识一条记录 |
| `transaction_id` |    是    | `6`字节  |         事务 ID         |
|  `roll_pointer`  |    是    | `7`字节  |        回滚指针         |

```
小贴士：实际上这几个列的真正名称其实是：DB_ROW_ID、DB_TRX_ID、DB_ROLL_PTR，我们为了美观才写成了row_id、transaction_id和roll_pointer。
```

&emsp;&emsp;这里需要提一下`InnoDB`表对主键的生成策略：优先使用用户自定义主键作为主键，如果用户没有定义主键，则选取一个`Unique`键作为主键，如果表中连`Unique`键都没有定义的话，则`InnoDB`会为表默认添加一个名为`row_id`的隐藏列作为主键。所以我们从上表中可以看出：<span style="color:red">InnoDB 存储引擎会为每条记录都添加 **_transaction_id_** 和 **_roll_pointer_** 这两个列，但是 **_row_id_** 是可选的（在没有自定义主键以及 Unique 键的情况下才会添加该列）</span>。这些隐藏列的值不用我们操心，`InnoDB`存储引擎会自己帮我们生成的。

&emsp;&emsp;因为表`record_format_demo`并没有定义主键，所以`MySQL`服务器会为每条记录增加上述的 3 个列。现在看一下加上`记录的真实数据`的两个记录长什么样吧：

![][04-11]

&emsp;&emsp;看这个图的时候我们需要注意几点：

1. 表`record_format_demo`使用的是`ascii`字符集，所以`0x61616161`就表示字符串`'aaaa'`，`0x626262`就表示字符串`'bbb'`，以此类推。

2. 注意第 1 条记录中`c3`列的值，它是`CHAR(10)`类型的，它实际存储的字符串是：`'cc'`，而`ascii`字符集中的字节表示是`'0x6363'`，虽然表示这个字符串只占用了 2 个字节，但整个`c3`列仍然占用了 10 个字节的空间，除真实数据以外的 8 个字节的统统都用<span style="color:red">空格字符</span>填充，空格字符在`ascii`字符集的表示就是`0x20`。

3. 注意第 2 条记录中`c3`和`c4`列的值都为`NULL`，它们被存储在了前面的`NULL值列表`处，在记录的真实数据处就不再冗余存储，从而节省存储空间。

#### CHAR(M)列的存储格式

&emsp;&emsp;`record_format_demo`表的`c1`、`c2`、`c4`列的类型是`VARCHAR(10)`，而`c3`列的类型是`CHAR(10)`，我们说在`Compact`行格式下只会把变长类型的列的长度<span style="color:red">逆序</span>存到`变长字段长度列表`中，就像这样：

![][04-12]

&emsp;&emsp;但是这只是因为我们的`record_format_demo`表采用的是`ascii`字符集，这个字符集是一个定长字符集，也就是说表示一个字符采用固定的一个字节，如果采用变长的字符集（也就是表示一个字符需要的字节数不确定，比如`gbk`表示一个字符要 1\~2 个字节、`utf8`表示一个字符要 1\~3 个字节等）的话，`c3`列的长度也会被存储到`变长字段长度列表`中，比如我们修改一下`record_format_demo`表的字符集：

```
mysql> ALTER TABLE record_format_demo MODIFY COLUMN c3 CHAR(10) CHARACTER SET utf8;
Query OK, 2 rows affected (0.02 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

&emsp;&emsp;修改该列字符集后记录的`变长字段长度列表`也发生了变化，如图：

![][04-13]

&emsp;&emsp;这就意味着：<span style="color:red">对于 **_CHAR(M)_** 类型的列来说，当列采用的是定长字符集时，该列占用的字节数不会被加到变长字段长度列表，而如果采用变长字符集时，该列占用的字节数也会被加到变长字段长度列表</span>。

&emsp;&emsp;另外有一点还需要注意，变长字符集的`CHAR(M)`类型的列要求至少占用`M`个字节，而`VARCHAR(M)`却没有这个要求。比方说对于使用`utf8`字符集的`CHAR(10)`的列来说，该列存储的数据字节长度的范围是 10 ～ 30 个字节。即使我们向该列中存储一个空字符串也会占用`10`个字节，这是怕将来更新该列的值的字节长度大于原有值的字节长度而小于 10 个字节时，可以在该记录处直接更新，而不是在存储空间中重新分配一个新的记录空间，导致原有的记录空间成为所谓的碎片。（这里你感受到设计`Compact`行格式的大佬既想节省存储空间，又不想更新`CHAR(M)`类型的列产生碎片时的纠结心情了吧。）

### Redundant 行格式

&emsp;&emsp;其实知道了`Compact`行格式之后，其他的行格式就是依葫芦画瓢了。我们现在要介绍的`Redundant`行格式是`MySQL5.0`之前用的一种行格式，也就是说它已经非常老了，但是本着知识完整性的角度还是要提一下，大家乐呵乐呵的看就好。

&emsp;&emsp;画个图展示一下`Redundant`行格式的全貌：

![][04-14]

&emsp;&emsp;现在我们把表`record_format_demo`的行格式修改为`Redundant`：

```
mysql> ALTER TABLE record_format_demo ROW_FORMAT=Redundant;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

&emsp;&emsp;为了方便大家理解和节省篇幅，我们直接把表`record_format_demo`在`Redundant`行格式下的两条记录的真实存储数据提供出来，之后我们着重分析两种行格式的不同即可。

![][04-15]

&emsp;&emsp;下面我们从各个方面看一下`Redundant`行格式有什么不同的地方：

- 字段长度偏移列表

  &emsp;&emsp;注意`Compact`行格式的开头是`变长字段长度列表`，而`Redundant`行格式的开头是`字段长度偏移列表`，与`变长字段长度列表`有两处不同：

  - 没有了<span style="color:red">变长</span>两个字，意味着`Redundant`行格式会把该条记录中<span style="color:red">所有列</span>（包括`隐藏列`）的长度信息都按照<span style="color:red">逆序</span>存储到`字段长度偏移列表`。

  - 多了个<span style="color:red">偏移</span>两个字，这意味着计算列值长度的方式不像`Compact`行格式那么直观，它是采用两个相邻数值的<span style="color:red">差值</span>来计算各个列值的长度。

    比如第一条记录的`字段长度偏移列表`就是：

    ```
    25 24 1A 17 13 0C 06
    ```

    因为它是逆序排放的，所以按照列的顺序排列就是：

    ```
    06 0C 13 17 1A 24 25
    ```

    按照两个相邻数值的<span style="color:red">差值</span>来计算各个列值的长度的意思就是：

    ```
    第一列(`row_id`)的长度就是 0x06个字节，也就是6个字节。
    第二列(`transaction_id`)的长度就是 (0x0C - 0x06)个字节，也就是6个字节。
    第三列(`roll_pointer`)的长度就是 (0x13 - 0x0C)个字节，也就是7个字节。
    第四列(`c1`)的长度就是 (0x17 - 0x13)个字节，也就是4个字节。
    第五列(`c2`)的长度就是 (0x1A - 0x17)个字节，也就是3个字节。
    第六列(`c3`)的长度就是 (0x24 - 0x1A)个字节，也就是10个字节。
    第七列(`c4`)的长度就是 (0x25 - 0x24)个字节，也就是1个字节。
    ```

- 记录头信息

  &emsp;&emsp;`Redundant`行格式的记录头信息占用`6`字节，`48`个二进制位，这些二进制位代表的意思如下：

|       名称        | 大小（单位：bit） |                                  描述                                  |
| :---------------: | :---------------: | :--------------------------------------------------------------------: |
|     `预留位1`     |        `1`        |                                没有使用                                |
|     `预留位2`     |        `1`        |                                没有使用                                |
|   `delete_mask`   |        `1`        |                          标记该记录是否被删除                          |
|  `min_rec_mask`   |        `1`        |             B+树的每层非叶子节点中的最小记录都会添加该标记             |
|     `n_owned`     |        `4`        |                        表示当前记录拥有的记录数                        |
|     `heap_no`     |       `13`        |                     表示当前记录在页面堆的位置信息                     |
|     `n_field`     |       `10`        |                           表示记录中列的数量                           |
| `1byte_offs_flag` |        `1`        | 标记字段长度偏移列表中每个列对应的偏移量是使用 1 字节还是 2 字节表示的 |
|   `next_record`   |       `16`        |                        表示下一条记录的相对位置                        |

&emsp;&emsp;第一条记录中的头信息是：

```
00 00 10 0F 00 BC
```

&emsp;&emsp;根据这六个字节可以计算出各个属性的值，如下：

```
预留位1：0x00
预留位2：0x00
delete_mask: 0x00
min_rec_mask: 0x00
n_owned: 0x00
heap_no: 0x02
n_field: 0x07
1byte_offs_flag: 0x01
next_record:0xBC
```

&emsp;&emsp;与`Compact`行格式的记录头信息对比来看，有两处不同： 1. Redundant 行格式多了 n_field 和 1byte_offs_flag 这两个属性。2. Redundant 行格式没有 record_type 这个属性。

- `1byte_offs_flag`的值是怎么选择的

  &emsp;&emsp;`字段长度偏移列表`实质上是存储每个列中的值占用的空间在`记录的真实数据`处结束的位置，还是拿`record_format_demo`第一条记录为例，`0x06`代表第一个列在`记录的真实数据`第 6 个字节处结束，`0x0C`代表第二个列在`记录的真实数据`第 12 个字节处结束，`0x13`代表第三个列在`记录的真实数据`第 19 个字节处结束，等等等等，最后一个列对应的偏移量值为`0x25`，也就意味着最后一个列在`记录的真实数据`第 37 个字节处结束，也就意味着整条记录的`真实数据`实际上占用`37`个字节。

  &emsp;&emsp;我们前面说过每个列对应的偏移量可以占用 1 个字节或者 2 个字节来存储，那到底什么时候用 1 个字节，什么时候用 2 个字节呢？其实是根据该条`Redundant`行格式`记录的真实数据`占用的总大小来判断的：

  - 当记录的真实数据占用的字节数不大于 127（十六进制`0x7F`，二进制`01111111`）时，每个列对应的偏移量占用 1 个字节。

  ```
  小贴士：如果整个记录的真实数据占用的存储空间都不大于127个字节，那么每个列对应的偏移量值肯定也就不大于127，也就可以使用1个字节来表示喽。
  ```

  - 当记录的真实数据占用的字节数大于 127，但不大于 32767（十六进制`0x7FFF`，二进制`0111111111111111`）时，每个列对应的偏移量占用 2 个字节。

  - 有没有记录的真实数据大于 32767 的情况呢？有，不过此时的记录已经存放到了溢出页中，在本页中只保留前`768`个字节和 20 个字节的溢出页面地址（当然这 20 个字节中还记录了一些别的信息）。因为`字段长度偏移列表`处只需要记录每个列在本页面中的偏移就好了，所以每个列使用 2 个字节来存储偏移量就够了。

  &emsp;&emsp;大家可以看出来，设计`Redundant`行格式的大佬还是比较简单粗暴的，直接使用整个`记录的真实数据`长度来决定使用 1 个字节还是 2 个字节存储列对应的偏移量。只要整条记录的真实数据占用的存储空间大小大于 127，即使第一个列的值占用存储空间小于 127，那对不起，也需要使用 2 个字节来表示该列对应的偏移量。简单粗暴，就是这么简单粗暴（所以这种行格式有些过时了～）。

  ```
  小贴士：大家有没有疑惑，一个字节能表示的范围是0～255，为什么在记录的真实数据占用的存储空间大于127时就采用2个字节表示各个列的偏移量呢？稍安勿躁，后边马上揭晓。
  ```

  &emsp;&emsp;为了在解析记录时知道每个列的偏移量是使用 1 个字节还是 2 个字节表示的，设计`Redundant`行格式的大佬特意在`记录头信息`里放置了一个称之为`1byte_offs_flag`的属性：

  - 当它的值为 1 时，表明使用 1 个字节存储。
  - 当它的值为 0 时，表明使用 2 个字节存储。

- `Redundant`行格式中`NULL`值的处理

  &emsp;&emsp;因为`Redundant`行格式并没有`NULL值列表`，所以设计`Redundant`行格式的大佬在`字段长度偏移列表`中的各个列对应的偏移量处做了一些特殊处理 —— 将列对应的偏移量值的第一个比特位作为是否为`NULL`的依据，该比特位也可以被称之为`NULL比特位`。也就是说在解析一条记录的某个列时，首先看一下该列对应的偏移量的`NULL比特位`是不是为`1`，如果为`1`，那么该列的值就是`NULL`，否则不是`NULL`。

  &emsp;&emsp;这也就解释了上面介绍为什么只要记录的真实数据大于 127（十六进制`0x7F`，二进制`01111111`）时，就采用 2 个字节来表示一个列对应的偏移量，主要是第一个比特位是所谓的`NULL比特位`，用来标记该列的值是否为`NULL`。

  &emsp;&emsp;但是还有一点要注意，对于值为`NULL`的列来说，该列的类型是否为定长类型决定了`NULL`值的实际存储方式，我们接下来分析一下`record_format_demo`表的第二条记录，它对应的`字段长度偏移列表`如下：

  ```
  A4 A4 1A 17 13 0C 06
  ```

  &emsp;&emsp;按照列的顺序排放就是：

  ```
  06 0C 13 17 1A A4 A4
  ```

  我们分情况看一下：

  - 如果存储`NULL`值的字段是定长类型的，比方说`CHAR(M)`数据类型的，则`NULL`值也将占用记录的真实数据部分，并把该字段对应的数据使用`0x00`字节填充。

    &emsp;&emsp;如图第二条记录的`c3`列的值是`NULL`，而`c3`列的类型是`CHAR(10)`，占用记录的真实数据部分 10 字节，所以我们看到在`Redundant`行格式中使用`0x00000000000000000000`来表示`NULL`值。

    &emsp;&emsp;另外，`c3`列对应的偏移量为`0xA4`，它对应的二进制实际是：`10100100`，可以看到最高位为`1`，意味着该列的值是`NULL`。将最高位去掉后的值变成了`0100100`，对应的十进制值为`36`，而`c2`列对应的偏移量为`0x1A`，也就是十进制的`26`。`36 - 26 = 10`，也就是说最终`c3`列占用的存储空间为 10 个字节。

  - 如果该存储`NULL`值的字段是变长数据类型的，则不在`记录的真实数据`处占用任何存储空间。

    &emsp;&emsp;比如`record_format_demo`表的`c4`列是`VARCHAR(10)`类型的，`VARCHAR(10)`是一个变长数据类型，`c4`列对应的偏移量为`0xA4`，与`c3`列对应的偏移量相同，这也就意味着它的值也为`NULL`，将`0xA4`的最高位去掉后对应的十进制值也是`36`，`36 - 36 = 0`，也就意味着`c4`列本身不占用任何`记录的实际数据`处的空间。

&emsp;&emsp;除了以上的几点之外，`Redundant`行格式和`Compact`行格式还是大致相同的。

#### CHAR(M)列的存储格式

&emsp;&emsp;我们知道`Compact`行格式在`CHAR(M)`类型的列中存储数据的时候还挺麻烦，分变长字符集和定长字符集的情况，而在`Redundant`行格式中十分干脆，不管该列使用的字符集是什么，只要是使用`CHAR(M)`类型，占用的真实数据空间就是该字符集表示一个字符最多需要的字节数和`M`的乘积。比方说使用`utf8`字符集的`CHAR(10)`类型的列占用的真实数据空间始终为`30`个字节，使用`gbk`字符集的`CHAR(10)`类型的列占用的真实数据空间始终为`20`个字节。由此可以看出来，使用`Redundant`行格式的`CHAR(M)`类型的列是不会产生碎片的。

### 行溢出数据

#### VARCHAR(M)最多能存储的数据

&emsp;&emsp;我们知道对于`VARCHAR(M)`类型的列最多可以占用`65535`个字节。其中的`M`代表该类型最多存储的字符数量，如果我们使用`ascii`字符集的话，一个字符就代表一个字节，我们看看`VARCHAR(65535)`是否可用：

```
mysql> CREATE TABLE varchar_size_demo(
    ->     c VARCHAR(65535)
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
mysql>
```

&emsp;&emsp;从报错信息里可以看出，`MySQL`对一条记录占用的最大存储空间是有限制的，除了`BLOB`或者`TEXT`类型的列之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过`65535`个字节。所以`MySQL`服务器建议我们把存储类型改为`TEXT`或者`BLOB`的类型。这个`65535`个字节除了列本身的数据之外，还包括一些其他的数据（`storage overhead`），比如说我们为了存储一个`VARCHAR(M)`类型的列，其实需要占用 3 部分存储空间：

- 真实数据
- 真实数据占用字节的长度
- `NULL`值标识，如果该列有`NOT NULL`属性则可以没有这部分存储空间

&emsp;&emsp;如果该`VARCHAR`类型的列没有`NOT NULL`属性，那最多只能存储`65532`个字节的数据，因为真实数据的长度可能占用 2 个字节，`NULL`值标识需要占用 1 个字节：

```
mysql> CREATE TABLE varchar_size_demo(
    ->      c VARCHAR(65532)
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
Query OK, 0 rows affected (0.02 sec)
```

&emsp;&emsp;如果`VARCHAR`类型的列有`NOT NULL`属性，那最多只能存储`65533`个字节的数据，因为真实数据的长度可能占用 2 个字节，不需要`NULL`值标识：

```
mysql> DROP TABLE varchar_size_demo;
Query OK, 0 rows affected (0.01 sec)

mysql> CREATE TABLE varchar_size_demo(
    ->      c VARCHAR(65533) NOT NULL
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
Query OK, 0 rows affected (0.02 sec)
```

&emsp;&emsp;如果`VARCHAR(M)`类型的列使用的不是`ascii`字符集，那会怎么样呢？来看一下：

```
mysql> DROP TABLE varchar_size_demo;
Query OK, 0 rows affected (0.00 sec)

mysql> CREATE TABLE varchar_size_demo(
    ->       c VARCHAR(65532)
    -> ) CHARSET=gbk ROW_FORMAT=Compact;
ERROR 1074 (42000): Column length too big for column 'c' (max = 32767); use BLOB or TEXT instead

mysql> CREATE TABLE varchar_size_demo(
    ->       c VARCHAR(65532)
    -> ) CHARSET=utf8 ROW_FORMAT=Compact;
ERROR 1074 (42000): Column length too big for column 'c' (max = 21845); use BLOB or TEXT instead
```

&emsp;&emsp;从执行结果中可以看出，如果`VARCHAR(M)`类型的列使用的不是`ascii`字符集，那`M`的最大取值取决于该字符集表示一个字符最多需要的字节数。在列的值允许为`NULL`的情况下，`gbk`字符集表示一个字符最多需要`2`个字节，那在该字符集下，`M`的最大取值就是`32766`（也就是：65532/2），也就是说最多能存储`32766`个字符；`utf8`字符集表示一个字符最多需要`3`个字节，那在该字符集下，`M`的最大取值就是`21844`，就是说最多能存储`21844`（也就是：65532/3）个字符。

```
小贴士：上述所言在列的值允许为NULL的情况下，gbk字符集下M的最大取值就是32766，utf8字符集下M的最大取值就是21844，这都是在表中只有一个字段的情况下说的，一定要记住一个行中的所有列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过65535个字节！
```

#### 记录中的数据太多产生的溢出

&emsp;&emsp;我们以`ascii`字符集下的`varchar_size_demo`表为例，插入一条记录：

```
mysql> CREATE TABLE varchar_size_demo(
    ->       c VARCHAR(65532)
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO varchar_size_demo(c) VALUES(REPEAT('a', 65532));
Query OK, 1 row affected (0.00 sec)
```

&emsp;&emsp;其中的`REPEAT('a', 65532)`是一个函数调用，它表示生成一个把字符`'a'`重复`65532`次的字符串。前面说过，`MySQL`中磁盘和内存交互的基本单位是`页`，也就是说`MySQL`是以`页`为基本单位来管理存储空间的，我们的记录都会被分配到某个`页`中存储。而一个页的大小一般是`16KB`，也就是`16384`字节，而一个`VARCHAR(M)`类型的列就最多可以存储`65532`个字节，这样就可能造成一个页存放不了一条记录的尴尬情况。

&emsp;&emsp;在`Compact`和`Reduntant`行格式中，对于占用存储空间非常大的列，在`记录的真实数据`处只会存储该列的一部分数据，把剩余的数据分散存储在几个其他的页中，然后`记录的真实数据`处用 20 个字节存储指向这些页的地址（当然这 20 个字节中还包括这些分散在其他页面中的数据的占用的字节数），从而可以找到剩余数据所在的页，如图所示：

![][04-16]

&emsp;&emsp;从图中可以看出来，对于`Compact`和`Reduntant`行格式来说，如果某一列中的数据非常多的话，在本记录的真实数据处只会存储该列的前`768`个字节的数据和一个指向其他页的地址，然后把剩下的数据存放到其他页中，这个过程也叫做`行溢出`，存储超出`768`字节的那些页面也被称为`溢出页`。画一个简图就是这样：

![][04-17]

&emsp;&emsp;最后需要注意的是，<span style="color:red">不只是 **_VARCHAR(M)_** 类型的列，其他的 **_TEXT_**、**_BLOB_** 类型的列在存储数据非常多的时候也会发生`行溢出`</span>。

#### 行溢出的临界点

&emsp;&emsp;那发生`行溢出`的临界点是什么呢？也就是说在列存储多少字节的数据时就会发生`行溢出`？

&emsp;&emsp;`MySQL`中规定<span style="color:red">一个页中至少存放两行记录</span>，至于为什么这么规定我们之后再说，现在看一下这个规定造成的影响。以上面的`varchar_size_demo`表为例，它只有一个列`c`，我们往这个表中插入两条记录，每条记录最少插入多少字节的数据才会`行溢出`的现象呢？这得分析一下页中的空间都是如何利用的。

- 每个页除了存放我们的记录以外，也需要存储一些额外的信息，乱七八糟的额外信息加起来需要`136`个字节的空间（现在只要知道这个数字就好了），其他的空间都可以被用来存储记录。

- 每个记录需要的额外信息是`27`字节。

  这 27 个字节包括下面这些部分：

  - 2 个字节用于存储真实数据的长度
  - 1 个字节用于存储列是否是 NULL 值
  - 5 个字节大小的头信息
  - 6 个字节的`row_id`列
  - 6 个字节的`transaction_id`列
  - 7 个字节的`roll_pointer`列

&emsp;&emsp;假设一个列中存储的数据字节数为 n，那么发生`行溢出`现象时需要满足这个式子：

```
136 + 2×(27 + n) > 16384
```

&emsp;&emsp;求解这个式子得出的解是：`n > 8098`。也就是说如果一个列中存储的数据不大于`8098`个字节，那就不会发生`行溢出`，否则就会发生`行溢出`。不过这个`8098`个字节的结论只是针对只有一个列的`varchar_size_demo`表来说的，如果表中有多个列，那上面的式子和结论都需要改一改了，所以重点就是：<span style="color:red">你不用关注这个临界点是什么，只要知道如果我们向一个行中存储了很大的数据时，可能发生`行溢出`的现象</span>。

### Dynamic 和 Compressed 行格式

&emsp;&emsp;下面要介绍另外两个行格式，`Dynamic`和`Compressed`行格式，我现在使用的`MySQL`版本是`5.7`，它的默认行格式就是`Dynamic`，这俩行格式和`Compact`行格式挺像，只不过在处理`行溢出`数据时有点儿分歧，它们不会在记录的真实数据处存储字段真实数据的前`768`个字节，而是把所有的字节都存储到其他页面中，只在记录的真实数据处存储其他页面的地址，就像这样：

![][04-18]

&emsp;&emsp;`Compressed`行格式和`Dynamic`不同的一点是，`Compressed`行格式会采用压缩算法对页面进行压缩，以节省空间。

## 总结

1. 页是`MySQL`中磁盘和内存交互的基本单位，也是`MySQL`是管理存储空间的基本单位。

2. 指定和修改行格式的语法如下：

   ```
   CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称

   ALTER TABLE 表名 ROW_FORMAT=行格式名称
   ```

3. `InnoDB`目前定义了 4 种行格式

   - COMPACT 行格式

     具体组成如图：
     ![][04-19]

   - Redundant 行格式
     具体组成如图：
     ![][04-20]
   - Dynamic 和 Compressed 行格式

     &emsp;&emsp;这两种行格式类似于`COMPACT行格式`，只不过在处理行溢出数据时有点儿分歧，它们不会在记录的真实数据处存储字符串的前 768 个字节，而是把所有的字节都存储到其他页面中，只在记录的真实数据处存储其他页面的地址。

     &emsp;&emsp;另外，`Compressed`行格式会采用压缩算法对页面进行压缩。

- 一个页一般是`16KB`，当记录中的数据太多，当前页放不下的时候，会把多余的数据存储到其他页中，这种现象称为`行溢出`。

  [04-01]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-01.png
  [04-02]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-02.png
  [04-03]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-03.png
  [04-04]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-04.png
  [04-05]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-05.png
  [04-06]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-06.png
  [04-07]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-07.png
  [04-08]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-08.png
  [04-09]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-09.png
  [04-10]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-10.png
  [04-11]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-11.png
  [04-12]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-12.png
  [04-13]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-13.png
  [04-14]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-14.png
  [04-15]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-15.png
  [04-16]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-16.png
  [04-17]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-17.png
  [04-18]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-18.png
  [04-19]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-19.png
  [04-20]: https://ngte-superbed.oss-cn-beijing.aliyuncs.com/book/mysql-learning-notes/04-20.png

<div STYLE="page-break-after: always;"></div>
