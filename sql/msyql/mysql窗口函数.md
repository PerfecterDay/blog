# 窗口函数
{docsify-updated}

> https://cloud.tencent.com/developer/article/2398505

窗口函数 =「在不减少行数的前提下，对一组“相关行”做计算」

## 什么是窗口函数
窗口函数（Window Functions）是SQL标准中的一个高级特性，它允许用户在不改变查询结果集行数的情况下，对每一行执行聚合计算或其他复杂的计算。这些计算**是基于当前行与结果集中其他行之间的关系进行的**。窗口函数特别适用于需要执行跨多行的计算，同时又想保持原始查询结果集的行数不变的场景。

1. 窗口函数的原理
窗口函数通过在查询结果集上定义一个“窗口”来工作，这个窗口可以是整个结果集，也可以是结果集的一个子集。窗口函数会对窗口内的行执行计算，并为每一行返回一个值。这个值是根据窗口内行的值以及窗口函数本身的逻辑计算得出的。

窗口函数不会改变查询结果集的行数，而是为每一行添加一个额外的列，这个列包含了窗口函数的计算结果。**这使得窗口函数非常适合于需要在保持原始数据的同时进行聚合或其他复杂计算的场景**。

2. 窗口函数的基本语法结构
```
<窗口函数>(<参数>) OVER (  
    [PARTITION BY <分区表达式>]  
    [ORDER BY <排序表达式> [ASC | DESC]]  
    [ROWS/Range <窗口范围>]  
)
```
+ `<窗口函数>(<参数>)` ：指定要使用的窗口函数及其参数。窗口函数可以是聚合函数（如SUM、AVG等），也可以是专门为窗口函数设计的函数（如ROW_NUMBER、RANK等）。
+ `OVER()` ：定义窗口的框架。所有窗口函数都需要使用OVER()子句来指定窗口的范围和行为。
+ `PARTITION BY <分区表达式>（可选）`：将结果集分成多个分区，窗口函数会在每个分区内独立执行。**分区表达式可以是一个或多个列名，用于确定如何将结果集分成不同的分区**。
+ `ORDER BY <排序表达式> ASC | DESC（可选）` ：指定窗口内行的排序顺序。排序表达式可以是一个或多个列名，用于确定窗口内行的排序方式。
+ `ROWS/Range <行范围>（可选）`：定义窗口的行范围。行范围可以是固定的行数（如 `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` ），也可以是相对于当前行的动态范围（如 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ，表示从窗口开始到当前行的所有行）。

```
select dname,ename,salary,row_number() over (partition by dname order by salary desc) as rk from employee;
select dname,ename,salary,rank() over (partition by dname order by salary desc) as rk from employee;
select dname,ename,salary,dense_rank() over (partition by dname order by salary desc) as rk from employee;
```

3. 窗口范围
MySQL的窗口函数中，指定窗口大小的语法主要是通过 `OVER()` 子句来实现的，其中可以使用 `ROWS` 或 `RANGE` 关键字来定义窗口的边界。不过，需要注意的是， `ROWS` 和 `RANGE` 定义了窗口的范围是基于物理行位置还是列值，而不是直接指定窗口的“大小”。窗口的“大小”实际上是由这些范围参数以及 `ORDER BY` 子句共同决定的。

```
OVER (  
    [PARTITION BY partition_expression, ... ]  
    [ORDER BY sort_expression [ASC | DESC], ...]  
    [ROWS frame_specification]  
    -- 或者  
    [RANGE frame_specification]  
)
```
其中， `frame_specification` 定义了窗口的起始和结束位置，它有以下几种形式：
```
BETWEEN frame_start AND frame_end ：指定窗口的开始和结束边界。
frame_start ：如果只指定了开始边界，则窗口会从该边界延伸到当前分区的最后一行。
frame_end ：通常不会只单独指定结束边界，因为它需要开始边界来形成完整的窗口范围。
```

对于ROWS和RANGE， `frame_start` 和 `frame_end` 可以是以下值之一：
```
UNBOUNDED PRECEDING：窗口从当前分区的第一行开始。
N PRECEDING：窗口从当前行之前的第N行开始，N是一个正整数。
CURRENT ROW：窗口从当前行开始。
N FOLLOWING：窗口从当前行之后的第N行开始。
UNBOUNDED FOLLOWING：窗口到当前分区的最后一行结束（通常只用于frame_end）。
```

<center><img src="pics/sql_frame,png" alt=""></center>

`ROWS` 是基于行的物理位置来确定窗口范围的，而 `RANGE` 则是基于 `ORDER BY` 子句中指定的列值来确定窗口范围的。 `RANGE` 在处理数值数据时特别有用，因为它可以包含与当前行值相近的其他行，即使它们的物理位置不相邻。

ROWS子句的常用选项:
```
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW：从窗口的开始到当前行。这是默认的窗口范围，如果未指定ROWS子句，则使用此范围。
ROWS BETWEEN N PRECEDING AND CURRENT ROW：从当前行之前的第N行到当前行。N必须是一个非负整数。
ROWS BETWEEN CURRENT ROW AND N FOLLOWING：从当前行到当前行之后的第N行。
ROWS BETWEEN N PRECEDING AND M FOLLOWING：从当前行之前的第N行到当前行之后的第M行。
```

RANGE子句的常用选项:
```
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW：从窗口开始到当前行值。具体大于还是小于当前值，与 order by 语句强关联，如果是 DESC 就是 >=当前值，如果是 ASC，就是 <= 当前行值
RANGE BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING：从当前行值到后续值。
RANGE BETWEEN N PRECEDING AND CURRENT ROW：从当前行值减去N到当前行值。这里的N通常是一个数字表达式，它指定了与当前行值相关的范围大小。
RANGE BETWEEN CURRENT ROW AND N FOLLOWING：从当前行值到当前行值加上N。
```

当没有指定 `rows` 或者 `range` 时，默认是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` .


## 窗口函数分类
1. 序号窗口函数
序号函数为结果集中的每一行分配一个唯一的序号或排名。这些函数通常基于排序顺序和其他条件来分配这些序号。

+ `ROW_NUMBER()` : 为每一行分配一个唯一的序号。
+ `RANK()` : 为每一行分配一个排名，对于相同的值会留下空位。
+ `DENSE_RANK()` : 为每一行分配一个排名，但不会为相同的值留下空位。

```
CREATE TABLE employees (  
    emp_id INT PRIMARY KEY,  
    emp_name VARCHAR(50),  
    salary DECIMAL(10, 2)  
);  
  
INSERT INTO employees (emp_id, emp_name, salary) VALUES  
(1, 'Alice', 50000),  
(2, 'Bob', 55000),  
(3, 'Charlie', 50000),  
(4, 'David', 60000),  
(5, 'Eva', 55000);

SELECT  
    emp_id,  
    emp_name,  
    salary,  
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,  
    RANK() OVER (ORDER BY salary DESC) AS rank,  
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank  
FROM  
    employees;
    
-- 这个查询的结果可能会是这样的：

emp_id | emp_name | salary | row_num | rank | dense_rank  
-------+----------+--------+---------+------+------------  
    4  |  David   | 60000  |    1    |  1   |     1  
    2  |   Bob    | 55000  |    2    |  2   |     2  
    5  |   Eva    | 55000  |    3    |  2   |     2  
    1  |  Alice   | 50000  |    4    |  4   |     3  
    3  | Charlie  | 50000  |    5    |  4   |     3

```

2. 分布窗口函数
分布函数用于计算值在窗口内的相对位置或分布。

+ `PERCENT_RANK()` : 计算行的百分比排名。
+ `CUME_DIST()` : 计算行相对于所有其他行的累积分布。

当使用窗口函数 `PERCENT_RANK()` 和 `CUME_DIST()` 时，这些函数通常用于计算结果集中行的相对排名和累积分布。

```
CREATE TABLE sales (  
    sale_id INT PRIMARY KEY,  
    sale_date DATE,  
    amount DECIMAL(10, 2)  
);  
  
INSERT INTO sales (sale_id, sale_date, amount) VALUES  
(1, '2023-01-01', 1000),  
(2, '2023-01-02', 1500),  
(3, '2023-01-03', 1200),  
(4, '2023-01-04', 1800),  
(5, '2023-01-05', 1100);


SELECT  
    sale_id,  
    sale_date,  
    amount,  
    PERCENT_RANK() OVER (ORDER BY amount DESC) AS percent_rank,  
    CUME_DIST() OVER (ORDER BY amount DESC) AS cume_dist  
FROM  
    sales;
-- 这个查询的结果可能会是这样的：

sale_id | sale_date   | amount | percent_rank | cume_dist  
--------+-------------+--------+--------------+-----------  
    4   | 2023-01-04  | 1800   |      0       |    0.2  
    2   | 2023-01-02  | 1500   |    0.25      |    0.4  
    3   | 2023-01-03  | 1200   |    0.5       |    0.6  
    5   | 2023-01-05  | 1100   |    0.75      |    0.8  
    1   | 2023-01-01  | 1000   |      1       |    1.0
```

3. 前后窗口函数
前后函数允许您访问与当前行相关的前一行或后一行的值。
+ `LAG(expr, offset, default)` : 返回指定偏移量之前的行的值。
+ `LEAD(expr, offset, default)` : 返回指定偏移量之后的行的值。

4. 首尾窗口函数
首尾函数允许您获取窗口的第一行或最后一行的值。
+ `FIRST_VALUE(expr)` : 返回窗口内第一行的值。
+ `LAST_VALUE(expr)` : 返回窗口内最后一行的值。

需要注意的是，FIRST_VALUE() 和 LAST_VALUE() 在没有指定 ORDER BY 子句时可能不会按预期工作，因为窗口的顺序是不确定的。此外，LAST_VALUE() 在某些情况下可能不如使用 LEAD() 函数灵活。

5. 聚合窗口函数
`SUM()` , `AVG()` ,  `MIN()` , `MAX()` 等也可以作为窗口函数使用，为每一行计算累计、移动或其他聚合值.