# 初级 SQL
{docsify-updated}

## 数据库表的新建修改与删除

1. 创建新的数据库：
`create databse [if not exists] db_name;`
2. 一个数据库服务器可以同时包含多个数据库，查看包含的数据库：
`show databases`
3. 删除数据库：
`drop database db_name`
1. 进入指定数据库：
`use db_name`
6. 查看数据库下有多少个表：
`show tables`
7. 查看某张表的结构：
`desc table_name`

常见的数据库对象：
<center><img src="pics/db-obj.jpg" width="70%"></center>

上述对象都可以在 create 、 drop 和 alter 等于语句中使用，如 create tabel.. 、 create index... 

Sql数据类型：
sql标准支持多种数据类型：
+ **char(n)**:**固定**长度的字符串，用户指定长度n，也可以使用全称character(n);
+ **varvhar(n)**:**可变长度**的字符串用户指定**最大长度n**，等价于全称character varying(n);
+ **int**:整数类型，等价于全称integer;
+ **smallint**:小整数类型;
+ **numeric(p,d)**:定点数，精度由用户指定。这个数有p位数字(包括一位符号位)，其中d位数字在小数点右边。所以一个numberic(3,1)可以精确表示44.5,但是不能精确表示444.5和0.32;
+ **real,double precision**:浮点数与双精度浮点数，精度与机器有关;
+ **float(n)**:精度至少为n位的浮点数。
每种类型都可能包含一个称作**空值**的特殊值 NULL 。

**char(10)** 类型的字段存储“Aou”三个字符时，不足10位的部分会补上7个空格来填充，而 **varchar(10)** 存储相同值时只会占用三个字符的空间，因此更推荐使用 **varchar(n)** 数据类型。

Mysql 的数据类型：
<center><img src="pics/mysql-types.jpg" width="60%"></center>

### 创建数据库表-mysql
1. 
```
creaet table [db-name].table-name(
    # 可以有多个如下的列定义
    column_name data_type(length) [NOT NULL] [DEFAULT value] [AUTO_INCREMENT] 
    ...,
    ...,
    primary key(column-name...)
) ENGINE=<engine-name> 
```

修改表的存储引擎： `ALTER TABLE 表名 ENGINE = 存储引擎名称;`

2. 从其他表中创建表并导入数据：
```
creaet table [db-name].table-name as select col1,col2... from table2
```
3. 创建一个与 table_name2 相同的表：  
```
create table table_name like table_name2
```

### 修改表结构

1. 增加列

```
alter table table-name add (
    # 可以有多个如下的列定义
    column-name datatype [default expr],
    ...
)

ALTER TABLE group_msg ADD effect_start_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP comment '开始生效时间' after state;
ALTER TABLE group_msg ADD expire_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP comment '失效时间' after effect_start_time;
```
实际上，如果表中已经有了数据，除非给新增字段指定默认值，否则新增字段不可以指定非空约束，因为新增字段肯定是空的。

2. 修改列定义：

```
alter table table-name modify column-name datatype [default expr] [first|after column-name2]

ALTER TABLE table_name RENAME COLUMN old_name TO new_name;
ALTER TABLE group_msg MODIFY effect_start_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP comment '开始生效时间' after state;
ALTER TABLE group_msg MODIFY expire_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP comment '失效时间' after effect_start_time;
```
first或 after column 代表将目标列修改到指定位置（column-name2 之前或之后）。如果数据表里已经有数据，修改列很容易失败，因为很可能修改的列定义与原有的数据记录有冲突。如果修改数据列的默认值，则只会对新增的数据有影响，已有的数据不会有任何影响。

3. 删除列
```
alter table table-name drop column-name
```
删除列总是可以成功的，且会从每行中删除该列的数据，并释放该列所占的空间。所以删除列一般时间比较长，因为还要回收空间。

### 删除表
    drop table table-name

### 清空表
    truncate table-name

## SLQ 查询

Sql 查询的语法顺序:
```
SELECT [DISTINCT]
FROM
WHERE
GROUP BY
HAVING
UNION
ORDER BY
LIMIT/ROWNUM
```
这 8 种关键字是有严格顺序的，
这 8 种关键字可以选择的写其中几种，但是顺序要按照上面的顺序来

SQL 查询的基本结构:  
一个sql查询包括三种类型子句： **select** 子句、 **from** 子句、 **where** 子句：

1. **select** 子句用于列出查询结果中所需要的属性
2. **from** 子句是一个查询求值中要访问的关系列表
3. **where** 子句是一个作用在 **from** 子句中关系上的谓词。
一个典型的sql查询具有如下形式：
<center><img src="pics/select-1.jpg" width=""></center>

每个 **A<sub>i</sub>** 代表一个属性，每个 **r<sub>i</sub>** 代表一个关系，P是一个谓词，如果省略 **where** 子句，那么谓词P为true。

通常来说，一个SQL查询的含义可以理解如下：

1. 为**from**子句中列出的关系产生笛卡尔积
   <center><img src="pics/dikaer.jpg" alt=""></center>
2. 在步骤1的基础上应用 **where** 子句中指定的谓词
3. 在步骤2结果中的每个元组，输出 **select** 子句中指定的属性(或表达式结果)。


### order by 
`select * from instructr order by salary desc, name asc`

查询 instructr 中所有元组，按照工资从高到低排序，如果两个元组工资相同，则按照名字升序排序。如果想在多个列上进行排序，必须对每一列指定 DESC/ASC 关键字,没有指定默认为ASC.

### limit
`limit 5 offset 6` 或者写成 `limit 6,5`
返回从第6行起（包括第6行）的5条数据。

### distinct
DISTINCT 关键字作用于所有的列，不仅仅是跟在其后的那一列，不能部分使用 DISTINCT。作用于多列时，必须所有列的值都相同才不会被重复检索出来。

### 集合运算
SQL 关系上的 `union 、 intersect 、 except` 运算对应于数学集合论中的并、交和差运算。三个运算都会自动去除输入中的重复。若想保留重复，分别在三个操作后加上 all 即可。  
`（select x from A) union [all] (select x from B)`

### 空值
如果算术表达式的任一输入为 null，则该算术表达式的结果为空。

涉及 null 的比较运算的结果为 unknown，是 true 和 false 之外的第三个逻辑值。
1. true and unknown = unknown；false and unknown = false；unknown and unknown = unknown
2. true or unknown = true；false or unknown = unknown；unknown or unknown = unknown
3. not unknown = unknown
如果 where 子句的谓词条件对一个元组计算出来的结果是 false 或 unknown，那么这个元组就不能被加入到结果集中。

可以用 `is null` 或者 `is not null` 来判断是否为空。不能用 =、!= 来判断。

### 通配符
+ %：表示任何字符出现任意次数
+ _：匹配单个任意字符，而不是多个字符

### 聚集函数
聚集函数是以值得一个集合为输入，返回单个值得函数。SQL提供了五个固有的聚集函数：
+ 平均值： avg
+ 最大值： max
+ 最小值： min
+ 总和：   sum
+ 计数：   count

### having子句
包含聚集、**group by**或**having**子句的查询的含义可通过下述操作序列来定义：

1. 与不带聚集的查询情况类似，最先根据 **from** 子句计算出一个关系； 
2. 如果出现 **where** 子句，**where** 子句中的谓词将应用到 **from** 子句的结果关系上； 
3. 如果出现了**group by**子句， 满足 **where** 谓词的元组通过**group by**子句形成分组。 如果没有 **group by** 子句，满足 **where** 谓词的整个元组集被当做一个分组。 
4. 如果出现了 **having** 子句，它将应用到每个分组上；不满足 **having** 子句谓词的分组将被抛弃。 
5. **select**子句 利用剩下的分组产生出查询结果中的元组，即在每个分组上应用聚集函数来得到单个结果元组。

出现在 **select** 语句中但没有被聚集的属性，只能是那些出现在 **group by** 子句中那些的属性。换句话说，任何没有出现在 **group by** 子句中的属性，如果出现在 **select** 子句中，**它只能出现在聚集函数内部**，否则这样的查询就是错误的。

与 **select** 子句类似, 任何出现在 **having** 子句中，但没有被聚集的属性必须出现在 **group by** 子句中，否则查询就被当成错误的。

### 对空值和布尔值的聚集
除了 **count(\*)** 外的所有聚集函数都**忽略**输入集合中的空值。

### case...when 语句
```
case
    when pred then result1
    when pred2 then result2
    ...
    when predn then resuln
    else result
end as column_name
```
统计指定条件下的数据(统计trade_account IS NOT NULL的总合)：
```
SELECT
COUNT(1) as total,
sum(
	case 
		when trade_account IS NOT NULL then 1
		else 0
	end
) as trade_count
FROM user;
```

### Like 
_和%字符在 LIKE 子句中具有特殊含义，用来匹配一个字符或任意一个字符序列 。 目前并不存在任何在宇面上使用它们的标准方式，所以如果想要匹配所有包含字符_的字符串，就必须使用下面的结构：
```... WHERE ? LIKE %!_% {escape ’!’} ```

!定义为转义字符，而`!_`组合表示字面常量下划线。

## SQL 删除
SQL删除的基本结构：
`delete from r wher p`
r代表一个关系，p代表一个谓词条件。首先从关系r中找出p为真的元组，然后将他们删除。

delete 语句只能用于一个关系，如果想从多个关系中删除，必须每个关系上使用一条 delete 命令。

## SQL 插入数据

```
insert into r(c1,c2,c3...)
values 
(value1,value2,value3...),
(value4,value5,value6...),
....
```
如果关系r后边没有指出具体的某些列，则默认插入所有列，需要为每列指定值。如果只为指定列赋值了，那么未赋值的列将被赋予空值 null。

## SQL 更新
```
update r
set c1=value1,c2=value2,...
where p
```
SQL首先检查关系中的所有元组，判断是否应该被更新，然后才执行更新。