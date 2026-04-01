# 读懂mysql执行计划
{docsify-updated}

-----------------------------------------------
https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain-join-types

------------------------
在MySQL使用 **explain** 关键字来查看SQL的执行计划，如下所示：
![mysql执行计划](../../pics/mysql-explain1.png)


#### id
有一组数字组成。表示一个查询中各个子查询的执行顺序：
+ id相同执行顺序由上至下
+ id不同，id值越大优先级越高，越先被执行。
+ id为null时表示一个结果集，不需要使用它查询，常出现在包含union等查询语句中。

#### select_type
每个子查询的查询类型，一些常见的查询类型。


select_type|description|
---|:--:|
SIMPLE|不包含任何子查询或union等查询的查寻|
PRIMARY|最外层查询就显示为 PRIMARY|
UNION|Second or later SELECT statement in a UNION|
DEPENDENT UNION	|Second or later SELECT statement in a UNION, dependent on outer query|
UNION RESULT|从UNION中获取结果集|
SUBQUERY|First SELECT in subquery|
DEPENDENT SUBQUERY	|First SELECT in subquery, dependent on outer query|
DERIVED|Derived table|
DEPENDENT DERIVED| Derived table dependent on another table|
MATERIALIZED|Materialized subquery|
UNCACHEABLE SUBQUERY|A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query|
UNCACHEABLE UNION|The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)|

#### table
查询的数据表，当从衍生表中查数据时会显示 `<derivedx> x` 表示对应的执行计划id。

#### partitions
表分区、表创建的时候可以指定通过那个列进行表分区。partitions这列表示查询将会匹配哪几个分区的行。

#### type
The type column of EXPLAIN output describes how tables are joined. In JSON-formatted output, these are found as values of the access_type property. The following list describes the join types, ordered from the best type to the worst:

##### 1. system
The table has only one row (= system table). This is a special case of the const join type.
表中只有一行。这是 const 类型的特殊情况

##### 2. const
The table has at most one matching row, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. const tables are very fast because they are read only once.

const is used when you compare all parts of a PRIMARY KEY or UNIQUE index to constant values. 

表中最多只有一条匹配的行（元组），一旦读取到，后续的行就不用了再去读取。
当用常量值去匹配所有主键列或唯一索引时（一个主键或唯一索引可以有多个列），就是 const 类型。

##### 3. eq_ref
从这张表中读取一行，用于前面表格的每行组合。 除了 system and const，这是最好的连接类型。 它在连接使用索引的所有部分时使用，索引是PRIMARY KEY或UNIQUE NOT NULL索引。

eq_ref可用于使用=运算符进行比较的索引列。 比较值可以是一个常量，也可以是一个表达式，该表达式使用在此表之前读取的表中的列。 在以下示例中，MySQL可以使用eq_ref连接来处理ref_table.

One row is read from this table for each combination of rows from the previous tables. Other than the system and const types, this is the best possible join type. It is used when all parts of an index are used by the join and the index is a PRIMARY KEY or UNIQUE NOT NULL index.

eq_ref can be used for indexed columns that are compared using the = operator. The comparison value can be a constant or an expression that uses columns from tables that are read before this table. In the following examples, MySQL can use an eq_ref join to process ref_table:
```
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```

##### 4. ref
All rows with matching index values are read from this table for each combination of rows from the previous tables. ref is used if the join uses only a leftmost prefix of the key or if the key is not a PRIMARY KEY or UNIQUE index (in other words, if the join cannot select a single row based on the key value). If the key that is used matches only a few rows, this is a good join type.

ref can be used for indexed columns that are compared using the = or <=> operator. In the following examples, MySQL can use a ref join to process ref_table:
```
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```
##### 5. fulltext
全文索引

##### 6. ref_or_null
This join type is like ref, but with the addition that MySQL does an extra search for rows that contain NULL values. This join type optimization is used most often in resolving subqueries. In the following examples, MySQL can use a ref_or_null join to process ref_table:
```
SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
```

##### 7. index_merge
This join type indicates that the Index Merge optimization is used. In this case, the key column in the output row contains a list of indexes used, and key_len contains a list of the longest key parts for the indexes used. For more information, see Section 8.2.1.3, “Index Merge Optimization”.

##### 8. unique_subquery
This type replaces eq_ref for some IN subqueries of the following form:
`value IN (SELECT primary_key FROM single_table WHERE some_expr)`
unique_subquery is just an index lookup function that replaces the subquery completely for better efficiency.

##### 9. index_subquery
This join type is similar to unique_subquery. It replaces IN subqueries, but it works for nonunique indexes in subqueries of the following form:
`value IN (SELECT key_column FROM single_table WHERE some_expr)`

##### 10. range
Only rows that are in a given range are retrieved, using an index to select the rows. The key column in the output row indicates which index is used. The key_len contains the longest key part that was used. The ref column is NULL for this type.

range can be used when a key column is compared to a constant using any of the =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, LIKE, or IN() operators:

```
SELECT * FROM tbl_name
  WHERE key_column = 10;

SELECT * FROM tbl_name
  WHERE key_column BETWEEN 10 and 20;

SELECT * FROM tbl_name
  WHERE key_column IN (10,20,30);

SELECT * FROM tbl_name
  WHERE key_part1 = 10 AND key_part2 IN (10,20,30);
```

##### 11. index
The index join type is the same as ALL, except that the index tree is scanned. This occurs two ways:

+ If the index is a covering index for the queries and can be used to satisfy all data required from the table, only the index tree is scanned. In this case, the Extra column says Using index. An index-only scan usually is faster than ALL because the size of the index usually is smaller than the table data.

+ A full table scan is performed using reads from the index to look up data rows in index order. Uses index does not appear in the Extra column.

MySQL can use this join type when the query uses only columns that are part of a single index.

##### 12. ALL
A full table scan is done for each combination of rows from the previous tables. This is normally not good if the table is the first table not marked const, and usually very bad in all other cases. Normally, you can avoid ALL by adding indexes that enable row retrieval from the table based on constant values or column values from earlier tables.

####  possible_keys
可能选择的索引,如果该列为 null 表示没有可用的索引。这通常是需要优化的。

#### key
实际使用的索引

#### key_len
索引的长度

#### ref
与使用的

#### rows
数据库预估的此次查询需要检索多少行数据。对于 InnoDB 来说，是个预估值，不代表最终真实扫描的行数。

#### filtered
按表条件过滤的行的百分比



