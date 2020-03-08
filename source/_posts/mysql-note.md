---
title: MySQL 学习笔记
author: 陈加菲
date: 2019-11-16 11:33:00
tags:
 - MySQL
---

最近看了尚硅谷 MySQL 视频，结合前段时间公司内部听的 MySQL 索引原理分享会，总结了一些 MySQL 的知识点，在这里整理输出，巩固记忆。
这篇文章会按如下五部分内容进行总结：

> 1. MySQL 介绍
> 2. SQL 执行顺序
> 3. 7 种 JOIN 类型
> 4. 索引
> 5. explain

## 01 MySQL 介绍

MySQL 是一个开源的关系型数据库，我们与数据库交互，一般的场景是把 SQL 发送给 MySQL，从 MySQL 中得到处理的结果。先来看一张架构图：

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/mysql-note/mysql-architecture.png" alt="MySQL 架构图" title="MySQL 架构图" style="zoom:120%;" />

由上至下依次为：

1. 连接层。这里主要跟客户端打交道，因此会暴露有不同语言的连接服务，同时维护客户端的连接池。
2. 服务层。这里有 SQL 处理接口，例如不同客户端传进来的 SQL 查询语句，会首先经过这里的处理，分析语句是否有问题，没问题的情况下，会被优化器进行优化，再向下层进行实际查询（或返回缓存的数据）。
3. 引擎层（可插拔式）。MySQL 有多种查询引擎，如 InnoDB、MyISAM 等。
4. 存储层。这里是磁盘上实际存储数据，元数据的文件系统。

在存储的数据中，主要分为**数据**，跟**日志**两大部分。

1. 数据文件：
   1. frm 文件，存储表结构
   2. myd 文件，存储数据
   3. myi 文件，存储索引
2. 日志文件：
   1. 二进制日志文件（bin-log），记录更改数据的语句，可根据该记录进行主从复制
   2. 错误日志（error-log），默认是关闭的，记录严重的警告和错误信息，每次启动和关闭的详细信息等
   3. 查询日志，默认是关闭的，记录查询的 SQL 语句，如果开启会降低 MySQL 的整体性能，因为记录日志也需要消耗系统性能

## 02 SQL 执行顺序

### 人写的顺序

先看看我们人是怎样编写查询 SQL 的：

``` sql
select distinct
	< select_list >
from < left_table >
< join_type > join < right_table > on < join_condition >
where 
	< where_condition >
group by
	< group_by_list >
having
	< having_condition >
order by
	< order_by_condition >
limit < limit_number >
```

上面 SQL 语句由上往下看出，我们写 SQL 语句，或者换种说法，给人看的 SQL 语句，依次为 

> 1. select 获取的字段
> 2. from 从哪张表加载数据来进行查询
> 3. where 条件筛选加载的数据
> 4. group by 对数据进行分组处理
> 5. having 进一步筛选分组后的数据
> 6. order by 对得到的数据进行排序
> 7. limit 获取指定条数的记录

但事实上，在 MySQL 处理中，是不按照上面的 7 个步骤依次执行的，在 01 MySQL 介绍中提到，MySQL 的服务层中有优化器，其作用是优化传进来的 SQL，以其规则执行 SQL 语句的处理。

### MySQL 执行的顺序

在 MySQL 中，处理查询 SQL 的顺序是这样的：

```sql
from < left_join >
on < join_condition >
< join_type > join < right_table >
where 
	< where_condition >
group by 
	< group_by_list >
having
	< having_condition >
select distinct
	< select_list >
order by 
	< order_by_condition >
limit < limit_number >
```

可以看出经优化器处理后 MySQL 执行 SQL 语句的顺序是

> 1. from 从哪张表加载数据来进行查询
> 2. where 条件筛选加载的数据
> 3. group by 对数据进行分组处理
> 4. having 进一步筛选分组后的数据
> 5. select 获取的字段
> 6. order by 对得到的数据进行排序
> 7. limit 获取指定条数的记录

此外，我们再看一张图，这张图说明了，我们在写 SQL 语句的每个步骤中，需要注意的问题：

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/mysql-note/sql-fishbone-illustration.png" alt="SQL 解析注意点鱼刺图" title="SQL 解析注意点鱼刺图" style="zoom:100%;" />

> 1. 在 from 加载表的时候，避免出现笛卡尔积
> 2. 联表加载数据，若是左右外连接，注意主表的全部数据会保留，同时应该使用小表作为主表，以小表驱动大表
> 3. where 条件筛选加载到内存中的数据，此时限制条件不能使用 select 字段的别名，只能使用表的原始字段名
> 4. 进行 group by 分组后，才能使用 having 筛选分组后的数据
> 5. order by 对筛选的数据进行排序，此时可以使用 select 字段的别名

## 03 7 种 JOIN 类型

SQL 联表中，主要有 7 中联表类型。

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/mysql-note/sql-join.jpg" alt="7 种 SQL JOIN 类型" title="7 种 SQL JOIN 类型" style="zoom:100%;" />

注意的一点是，MySQL 中不支持 full outer join，因此要实现 full outer join（全连接），可使用 union（或 union all）代替。

## 04 索引

### 索引是什么

MySQL 官方对索引的定义是：

> 索引（Index）是帮助 MySQL 高效获取数据的**数据结构**

因此索引的本质是数据结构，其目的在于提高查询效率，简单理解便是排好序的快速查找数据结构。

一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件（myi 文件）的形式存储在硬盘上。因此我们简单总结出索引的优劣势

> 优势：加快查询速度
>
> 劣势：插入，更新，删除数据时需要更新索引来维护索引的准确，降低了修改数据的效率

### Innodb 使用的 B+ 树索引

B+ 树的特点是**数据地址都在叶子节点（查询速度稳定）、叶子节点排序、叶子节点间有指针相连（范围查询）**。

### 索引类型

依据 B+ 树的数据结构，在 Innodb 中有几种不同的索引类型。

1）聚簇索引，也就是**主键索引**，是对表中主键 ID 的索引，其树的叶子节点直接指向硬盘上的实际数据。

为什么叫聚簇？“聚簇” 表示数据行按 ID 顺序存储在一起，且聚簇索引的顺序就是数据存储的物理顺序。ID 顺序读取性能最高，磁盘 IO 次数最少。由于数据在物理存储中只能有一个顺序，所以在一个表中只能有一个聚簇索引 。

> 在 Innodb 引擎中，如果一张数据表声明了主键，那么主键列会作为聚簇索引。如果没有主键，则会选择一个唯一且不为空的索引列作为主键，成为聚簇索引。如果前面两个要求都不符合，Innodb 会自动生成一个隐含字段作为主键。

聚簇索引只是主键，那么其他类型的索引呢？

2）二级索引，其树的叶子节点指向主键 ID 的值。

二级索引的叶子节点的数据域存储的是主键值而不是具体数据行地址。通过二级索引查找行，首先要在二级索引中找到对应行的主键，然后用该主键在主键索引中找到对应的数据行。**因此用二级索引查找需要经历两次 B+ 树的查找**。

3）联合索引

例如一张表有 school, grade, subject 三个字段，共同组成一个联合索引。一个联合索引是一个多元组，例如这里是一个 (school, grade, subject) 的三元组，在索引结构中，会从组成索引的字段中，**从左往右依次排序**。因此**这里会先按照 school 排序，在排好序的数据中再根据 grade 排序，最后再根据 subject 排序**。

只有理解好这种排序规则，我们便能明白，在 where 限制条件中，若没有命中 school（没有 school 的筛选条件），就算有 grade 跟 subject 的筛选，上述联合索引生成的结构对查询是没有帮助的。要想使用到上述生成的联合索引，就首先要有 school 的筛选，这就是**最左匹配原则**。

下面再使用 SQL 例子介绍

```sql
... where school = ? and grade = ? -- 可以匹配联合索引
... where grade = ? and subject = ? -- 不可以匹配联合索引
... where grade = ? and subject = ? and school = ? -- 可以匹配联合索引，优化器会帮我们调整限制条件的顺序，所以只要限制条件中存在联合索引的最左列就可以，与顺序无关
```

4）覆盖索引，select 的值是索引叶子节点的值，不需要去查硬盘上的具体数据。

### 索引的注意事项

1. 索引的建立需具体需求具体分析，单表索引不宜过多，因为会影响写入性能。

2. 列值区分度要高（表示字段不重复的比例要高），如使用下面的公式简单计算列值：

   ```sql
   count(distinct col)/count(*)
   ```

3. 索引列不能参与函数计算。

4. where 限制条件中不能同时使用多个索引，因为每次使用一个索引（除聚簇索引，覆盖索引）便会查询到所要数据的主键 ID 值，此时便会走主键索引在硬盘上获取数据。


## 05 explain

使用 SQL 查询语句，怎么知道有没有命中索引？我们可以通过 explain 关键字来对 SQL 语句进行分析。

explain 分析能让我们知道：

- 表的读取顺序
- 数据读取操作的操作类型
- 哪些索引可以使用
- 哪些索引被实际使用
- 表之间的引用
- 每张表有多少行被优化器查询

运行 explain，可以获取如下多列的信息，这些信息包含了上面几个方面的内容

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/mysql-note/explain-column.png" alt="explain 分析列" title="explain 分析列" style="zoom:100%;" />

### 1）id

select 查询的序列号，包含一组数字，表示查询中执行 select 子句或操作表的顺序。**可以解决表的读取顺序问题**。

> 1）id 相同的情况下，执行顺序从上往下执行
>
> 2）如果是子查询，id 的序号会递增，id 的值越大优先级越高，越先被执行
>
> 3）id 如果相同，可以认为是一组，从上往下执行；在所有组中，id 值越大，优先级越高，越先执行

### 2）select_type

select_type 常见的有 6 种：**SIMPLE / PRIMARY / SUBQUERY / DERIVED / UNION / UNION RESULT**

其主要是用来告诉我们查询的类型，用于区别普通查询、联合查询、子查询等的复杂查询。

> 1）SIMPLE：简单的 select 查询，查询中不包含子查询或者 UNION
>
> 2）PRIMARY：查询中若包含任何复杂的子部分，最外层查询则被标记为 PRIMARY（想象成一个鸡蛋的鸡蛋壳）
>
> 3）SUBQUERY：在 select 或 where 列表中包含了子查询
>
> 4）DERIVED：在 from 列表中包含的子查询被标记为 DERIVED（衍生），MySQL 会递归执行这些子查询，把结果放在临时表里
>
> 5）UNION：若第二个 select 出现在 UNION 之后，则被标记为 UNION；若 UNION 包含在 from 子句的子查询中，外层 select 将被标记为：DERIVED
>
> 6）UNION RESULT：从 UNION 表获取结果的 select

### 3）table

显示这一行的数据是关于哪张表的

### 4）type

表示访问类型，显示查询使用了哪些类型，从好到差依次是：

> system > const > eq_ref > ref > range > index > ALL

一般来说要优化到 range 级别，最好是到 ref 级别。

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/mysql-note/explain-type.png" alt="type 类型" title="type 类型" style="zoom:100%;" />

> 1）system：表只有一行记录（等于系统表），这是 const 类型的特例，平时不会出现，这个可以忽略不计
>
> 2）const：表示通过索引一次就找到了，const 用于比较 primary key 或者 unique 索引。因为只匹配一行数据，所以很快，如将主键置于 where 列表中，MySQL 就能将该查询转换成一个常量
>
> 3）eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一性索引
>
> 4）ref：非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而，它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体
>
> 5）range：只检索给定范围的行，使用一个索引来选择行。key 列显示使用了哪个索引，一般就是在你的 where 语句中出现了 between、<、>、in 等的查询。这种范围扫描查询比全表扫描要好，因为它只需要开始于索引的某一点，而结束于另一点，不用扫描全部索引
>
> 6）index：Full Index Scan，index 与 ALL 区别为 index 类型只遍历索引树，这通常比 ALL 快，因为索引文件通常比数据文件小。（也就是说虽然 ALL 和 index 都是读全表，但 index 是从索引中读取，而 ALL 是从硬盘中读取的）
>
> 7）ALL：Full Table Scan，将遍历全表以找到匹配的行

注意，const 跟 eq_ref 都是唯一性扫描，其区别是：const 是**直接**按主键或唯一键**读取**，eq_ref 用于联表查询的情况，按**联表**的主键或唯一键联合**查询**。

### 5）possible keys

显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。

### 6）key

实际使用的索引。如果为 null，则没有使用索引。查询中若使用了覆盖索引（可见上方 04)索引 介绍），则该索引仅出现在 key 列表中。

### 7）ken_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的前提下，长度越短越好。

key_len 显示的值为索引字段的最大可能长度，**并非实际使用长度**，即 key_len 是根据表定义计算而得，不是通过表内检索出的。

### 8）ref

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值。

### 9）rows

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数。

### 10）Extra

包含不适合在其他列中显示但十分重要的额外信息。其主要包括如下几种情况：

> 1）Using  filesort（重要）：说明 MySQL 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL 中无法利用索引完成的排序操作称为 “文件排序”（这是比较严重性能的情况）
>
> 2）Using temporary（重要）：使用了临时表保存中间结果，MySQL 在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by（这也是影响性能的情况）
>
> 3）Using index（重要）：表示相应的 select 操作中使用了覆盖索引（Covering Index，见上方 04)索引 介绍），避免访问了表的数据行。（出现这种情况表示效率不错）
>
> - 如果同时出现 Using where，表明索引被用来执行索引键值的查找；
> - 如果没有同时出现 Using where，表明索引用来读取数据而非执行查找动作。
>
> 4）Using where：表明使用了 where 过滤
>
> 5）Using join buffer：使用连接缓存
>
> 6）Impossible WHERE：where 子句的值总是 false，不能用来获取任何元祖
>
> 7）select tables optimized away（不多）：在没有 group by 子句的情况下，基于索引优化 MIN/MAX 操作或者对于 MyISAM 存储引擎优化 count(*) 操作，不必等到执行阶段再进行计算，查询执行计划生成阶段即完成优化。 
>
> 8）distinct（不多）：优化 distinct 操作，在找到第一匹配的元祖后即停止找同样值的动作



## 参考文章

[mysql（1）—— 详解一条sql语句的执行过程](https://www.cnblogs.com/cdf-opensource-007/p/6502556.html)
[浅析MySQL InnoDB中的B+树索引](https://juejin.im/post/5c3b22a1f265da61776c2bf2)
[mysql const与eq_ref的区别](https://www.cnblogs.com/maohuidong/p/10491563.html)
[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)




**END**