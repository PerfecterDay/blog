# 执行计划
{docsify-updated}

+ `id`: 在一个大的查询语句中每个 SELECT 关键字都对应一个唯一的 id
+ `select_type`: SELECT 关键字对应的那个查询的类型
+ `table`: 表名
+ `partitions`: 匹配的分区信息
+ `type`: 针对单表的访问方法
+ `possible_keys`: 可能用到的索引
+ `key`: 实际上使用的索引
+ `key_len`: 实际使用到的索引长度
+ `ref`: 当使用索引列等值查询时，与索引列进行等值匹配的对象信息
+ `rows`: 预估的需要读取的记录条数
+ `filtered`: 某个表经过搜索条件过滤后剩余记录条数的百分比
+ `Extra`: 一些额外的信息


## id
查询询语句一般都以 `SELECT` 关键字开头，  比较简单的查询语句中只有一个 `SELECT` 关键字，但是在下面这两种情况下， 一条查询语句中会出现多个 `SELECT` 关键字 :

1. 查询中包含子查询的情况。 比如下面这个查询语句中就包含 2 个 `SELECT` 关键字:
    ```sql
    SELECT * FROM s1 WHERE key1 IN (SELECT key3 FROM s2);
    ```
2. 查询 中包含 `UNION` 子句的情况。 比如下面这个查询语句 中 就包含 2 个 SELECT 关键字:
   ```sql
    SELECT * FROM s1 UNION SELECT * FROM s2;
   ```

查询语句中每出现一个 `SELECT` 关键字 ， 设计 MySQL 的 大叔就会为它分配一个唯一的ID值 ， 这个 id 值就是 `EXPLAIN` 时输出的第一列 。

对于连接查询来说， 一个 `SELECT` 关键字后面的 `FROM` 子句中可以跟随多个表。在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的，出现在前边的表表示驱动表，出现在后边的表表示被驱动表。

在许多 UNION 语句中，它的执行计划可能还包含 ID 为 NULL 的条目，íd 为 NULL 表明这个临时表是为了合并两个查询的结果集而创建的。

+ id相同执行顺序由上至下
+ id不同，id值越大优先级越高，越先被执行。
+ id为null时表示一个结果集，不需要使用它查询，常出现在包含union等查询语句中。

## select_type
一条大的查询语句里边可以包含若干个 SELECT 关键字，每个 SELECT 关键字代表着一个小的查询语句，而每个 SELECT 关键字的 FROM 子句中都可以包含若干张表（这些表用来做连接查询），每一张表都对应着执行计划输出中的一条记录，对于在同一个 SELECT 关键字中的表来说，它们的 id 值是相同的。

设计 MySQL 的大叔为每一个 SELECT 关键字代表的小查询都定义了一个称之为 select_type 的属性，意思是我们只要知道了某个小查询的 select_type 属性，就知道了这个小查询在整个大查询中扮演了一个什么角色，我们还是先来见识见识这个 `select_type` 都能取哪些值:
+ `SIMPLE`: 查询语句中不包含 `UNION` 或者子查询的查询都算作是 `SIMPLE` 类型
+ `PRIMARY`: 对于包含 `UNION` 、 `UNION ALL` 或者子查询的大查询来说，它是由几个小查询组成的，其中最左边的那个查询的 select_type 值就是 `PRIMARY`
+ `UNION`: 对于包含 `UNION` 或者 `UNION ALL` 的大查询来说，它是由几个小查询组成的，其中除了最左边的那个小查询以外，其余的小查询的 select_type 值就是 `UNION`
+ `UNION RESULT`: MySQL 选择使用临时表来完成 UNION 查询的去重工作，针对该临时表的查询的 select_type 就是 `UNION RESULT`
+ `SUBQUERY`: 如果包含子查询的查询语句不能够转为对应的 semi-join 的形式，并且该子查询是不相关子查询，并且查询优化器决定采用将该子查询物化的方案来执行该子查询时，该子查询的第一个 SELECT 关键字代表的那个查询的 select_type 就是 `SUBQUERY`
+ `DEPENDENT SUBQUERY`: 如果包含子查询的查询语句不能够转为对应的 semi-join 的形式，并且该子查询是相关子查询，则该子查询的第一个 SELECT 关键字代表的那个查询的 select_type 就是 `DEPENDENT SUBQUERY`。select_type为`DEPENDENT SUBQUERY`的查询可能会被执行多次
+ `DEPENDENT UNION`: 在包含 UNION 或者 UNION ALL 的大查询中，如果各个小查询都依赖于外层查询的话，那除了最左边的那个小查询之外，其余的小查询的 select_type 的值就是 `DEPENDENT UNION`
+ `DERIVED`: 对于采用物化的方式执行的包含派生表的查询，该派生表对应的子查询的 select_type 就是 `DERIVED`
+ `MATERIALIZED`: 当查询优化器在执行包含子查询的语句时，选择将子查询物化之后与外层查询进行连接查询时，该子查询对应的 select_type 属性就是 `MATERIALIZED` 
+ `UNCACHEABLE SUBQUERY`: A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query
+ `UNCACHEABLE UNION`: The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)

## partitions
分区

## type
执行计划的一条记录就代表着 MySQL 对某个表的执行查询时的访问方法，其中的 type 列就表明了这个访问方法是个啥。完整的访问方法如下： `system` ， `const` ， `eq_ref` ， `ref` ， `fulltext` ， `ref_or_null` ， `index_merge` `，unique_subquery` ， `index_subquery` `，range` ， `index` ， `ALL` 。


## possible_keys和key
`possible_keys` 列表示在某个查询语句中，对某个表执行单表查询时可能用到的索引有哪些， key 列表示实际用到的索引有哪些。 `possible_keys` 列中的值并不是越多越好，可能使用的索引越多，查询优化器计算查询成本时就得花费更长时间，所以如果可以的话，尽量删除那些用不到的索引。

## key_len
key_len 列表示当优化器决定使用某个索引执行查询时，该索引记录的最大长度，它是由这三个部分构成的：
+ 对于使用固定长度类型的索引列来说，它实际占用的存储空间的最大长度就是该固定值，对于指定字符集的
+ 变长类型的索引列来说，比如某个索引列的类型是 VARCHAR(100) ，使用的字符集是 utf8 ，那么该列实际占
+ 用的最大存储空间就是 100 × 3 = 300 个字节。
+ 如果该索引列可以存储 NULL 值，则 key_len 比不可以存储 NULL 值时多1个字节。
+ 对于变长字段来说，都会有2个字节的空间来存储该变长列的实际长度。

## ref
当使用索引列等值匹配的条件去执行查询时，也就是在访问方法是 `const` 、 `eq_ref` 、 `ref` 、 `ref_or_null` 、 `unique_subquery` 、 `index_subquery` 其中之一时， ref 列展示的就是与索引列作等值匹配的东东是个啥，比如只是一个常数或者是某个列 。

## rows
如果查询优化器决定使用全表扫描的方式对某个表执行查询时，执行计划的 rows 列就代表预计需要扫描的行数，如果使用索引来执行查询时，执行计划的 rows 列就代表预计扫描的索引记录行数。

## filtered
之前在分析连接查询的成本时提出过一个 condition filtering 的概念，就是 MySQL 在计算驱动表扇出时采用的一个策略：
+ 如果使用的是全表扫描的方式执行的单表查询，那么计算驱动表扇出时需要估计出满足搜索条件的记录到底有多少条。
+ 如果使用的是索引执行的单表扫描，那么计算驱动表扇出的时候需要估计出满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

## Extra
Extra 列是用来说明一些额外信息的，我们可以通过这些额外信息来更准确的理解 MySQL 到底将如何执行给定的查询语句。 MySQL 提供的额外信息有好几十个，我们就不一个一个介绍了，所以我们只挑一些平时常见的或者比较重要的额外信息介绍。

+ `No tables used`: 当查询语句的没有 FROM 子句时将会提示该额外信息
+ `Impossible WHERE`: 查询语句的 WHERE 子句永远为 FALSE 时将会提示该额外信息
+ `No matching min/max row`: 当查询列表处有 MIN 或者 MAX 聚集函数，但是并没有符合 WHERE 子句中的搜索条件的记录时，将会提示该额外信息
+ `Using index`: 当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用索引覆盖的情况下，在Extra 列将会提示该额外信息
+ `Using index condition`: 如果在查询语句的执行过程中将要使用 索引条件下推 这个特性，在 Extra 列中将会显示该额外信息
+ `Using where`: 当我们使用全表扫描来执行对某个表的查询，并且该语句的 WHERE 子句中有针对该表的搜索条件时，在Extra 列中会提示上述额外信息
+ `Using join buffer (Block Nested Loop)`: 在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度， MySQL 一般会为其分配一块名叫 join buffer 的内存块来加快查询速度，也就是我们所讲的 基于块的嵌套循环算法
+ `Not exists`: 当我们使用左（外）连接时，如果 WHERE 子句中包含要求被驱动表的某个列等于 NULL 值的搜索条件，而且那个列又是不允许存储 NULL 值的，那么在该表的执行计划的 Extra 列就会提示 Not exists 额外信息
+ `Using filesort`: 如果某个查询需要使用**文件排序**的方式执行查询，就会在执行计划的 Extra 列中显示该额外信息
+ `Using temporary`: 使用临时表