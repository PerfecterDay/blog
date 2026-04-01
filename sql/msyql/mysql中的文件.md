# mysql 中的文件
{docsify-updated}


#### 文件种类
1. 配置参数文件：告诉mysql在启动时在哪里可以找到数据库文件，并且指定一些初始化参数配置来数据库。
2. 日志文件：错误日志文件、二进制日志文件、慢查询日志文件、查询日志文件等。
3. socket 文件：使用 UNIX 套接字连接时需要的文件
4. pid文件：mysql 实例的进程 ID文件
5. mysql表结构文件：用来存放 Mysql表结构定义文件。
6. 存储引擎文件：Mysql支持多种存储引擎，每种存储引擎都会有自己的文件来存储各种数据。

#### 参数文件
命令行中输入 `mysql --help | grep my.cnf` 可以查看 mysql 查找配置文件路径的顺序。
mysql 参数有 **静态参数** 和 **动态参数** 之分：
1. 动态参数意味着在mysql运行过程中能够动态的更改。
2. 静态参数在实例运行后便不可更改。
使用 **set** 命令可以对动态的参数值进行修改：
```
set [global | session ] system_var_name=expr
set [@@global. | @@session. | @@]system_var_name=expr
```
有些参数只能在会话中修改，有些参数可以在整个生命周期中都会生效，有些参数既可以在会话中生效又可以在整个实例的生命周期内生效。
使用 `show variables like 'xxx'` 可以查看参数配置的值。

#### 日志文件
1. 错误日志  
   错误日志文件对mysql的启动、运行和关闭过程进行记录，其实就是 mysql 运行的日志文件。可以使用命令 `show variables like 'log_error'` 来定位改文件。
2. 慢查询日志  
    可以在mysql启动时设定一个阈值，将运行时间超过（大于）该阈值的所有 SQL 语句都记录到慢查询日志文件中。DBA可以对这些语句进行优化。该阈值可以通过 `long_query_time` (mysql8 中是`slow_query_log`)参数来设置，默认是10秒。通过 `slow_query_log_file`设置慢查询日志的位置。
    另一个和慢查询有关的参数是 `log_queries_not_using_indexes`, 如果这个参数为 on 的话，那么没有使用索引的sql语句也会被记录到慢查询日志中。`log_throttle_queries_not_using_indexes` 用来设置每分钟允许记录到慢查询日志中的未使用索引的sql语句的次数，默认是0表示没有限制。生产环境中建议设置以防止慢查询日志不断快速增加。
    慢查询日志变得很大时，可以使用 `mysqldumpslow` 命令来分析慢查询日志。
3. 查询日志  
    查询日志记录了所有的数据库查询请求信息，无论这些查询是否得到了正确的执行。`general_log`和 `general_log_file`分别设置是否开启查询日志以及日志文件的存储位置。
4. 二进制日志  
    二进制日志记录了所有的对数据库更改的操作日志，不包括 Select/Show这类的操作，因为这类操作不会改变数据库中的数据。但是有些 DML（insert、delete、update） 语句即使没有真正的改变数据，可能也会被记录到日志中。二进制日志有以下几种作用：
    1. 恢复:可以通过二进制日志进行 point-in-time 的恢复
    2. 复制：通过复制二进制日志到另一台数据库恢复即可实现复制
    3. 审计：通过二进制日志的信息进行审计来判断是否有注入攻击
     
    参数 `log_bin` 、 `log_bin_basename` 和 `log_bin_index` 分别用来设置是否打开二进制日志、二进制日志的基础文件名和二进制日志索引文件名，以下参数影响二进制日志的行为：
    + `max_binlog_size`:单个日志文件的最大大小
    + `binlog_cache_size`：基于会话的二进制日志缓冲区（每个会话一个缓冲区），未提交的二进制日志直接写入缓冲区，事务提交时将缓冲区内容写入二进制日志，改参数指定缓冲区大小。如果写入日志超出缓冲区大小，会写入一个临时文件。
    + `sync_binlog`：表示写缓冲区多少次后写入到磁盘文件，如果为1表示同步写磁盘，这样会损失一定性能但是可以获得最高的可用性。否则写入缓冲区的文件在宕机有可能未刷盘造成丢失。
    + `binlog_format`：设置二进制日志文件的格式
        1. statement: 记录的是数据库的逻辑 SQL 语句,执行update一个值=1 1000次，会产生1000条记录。
        2. row: 记录的是数据库表的行更改情况
        3. mixed: 默认statement格式，但是某些情况下使用 row 格式。执行update一个值=1 1000次，只会产生1条记录
   
    二进制日志文件不能用 cat、less等命令查看，必须使用 `mysqlbinlog` 工具来查看，statement 格式的日志直接能看到逻辑SQL语句，而 row 格式的日志需要加上 `-v` 或 `-vv` 参数。遇到`mysqlbinlog: [ERROR] unknown variable 'default-character-set=utf8'`错误时，使用 `mysqlbinlog --no-defaults -v binlog.000109` 即可。

### 套接字文件
在 UNIX 系统下本地连接 mysql 可以使用 UNIX 域套接字方式，这种方式需要一个套接字文件，使用 `show variables like '%socket%'` 可以查看配置。

### PID文件
mysql 启动时会将进程ID写入一个文件，改文件就是 pid 文件。使用 `show variables like 'pid_file'` 查看配置。

### 表结构定义文件
mysql是插件式的存储引擎结构，但无论一张表采取那种存储引擎,Mysql都有一个以frm结尾的后缀名文件，这个文件记录了表结构定义。视图也会有一个frm文件定义视图结构。Mysql8版本删除了 frm 表结构文件，将表结构直接存放在 ibd 文件中。

### innodb存储引擎文件
主要介绍表空间文件和重做日志文件。

#### 表空间文件
Innodb 存储引擎以索引组织表的形式来存储数据，所有的数据都存放在一个空间中，称为表空间。表空间又由段、区、页组成。大致如下图所示：
<center><img src="/pics/mysql-tablespace.png" width="50%"></center>

innodb 采用将存储数据按表空间进行存放的设计，默认配置下会有一个初始大小10M的名为 ibdata1 的文件，该文件就是默认的表空间文件。用户可以通过参数 `innodb_data_file_path` 对默认表空间文件进行设置。设置表空间文件后，所有基于 innodb 存储引擎的表的数据都会保存到该共享表空间中。

若设置了 `innodb_file_per_table = on`, mysql 会为每张基于 innodb 引擎的表产生独立的表空间文件，命名规则为：表名.ibd。这些单独的表空间文件仅存储该表的**数据、索引和插入缓冲BITMAP**等信息，其余信息如回滚信息（UNDO）、插入缓冲索引页、系统事务信息、双写缓冲等还是存在共享表空间中。

##### 分区表
+ 查看分区：
    1. `show create table REGSAPICallAudit`
    2. 查看表是否采用了分区:`show table status like 'regsapicallaudit'\G`
    3. 查看 `information_schema.partitions` 表可以查看详细分区信息：
   ```
    select
    partition_name part,  
    partition_expression expr,  
    partition_description descr,  
    table_rows  
    from information_schema.partitions  where 
    table_schema = schema()  
    and table_name='tr';
    ```
+ 添加分区：  `ALTER TABLE REGSAPICallAudit ADD PARTITION (PARTITION` \`p\${date}\` `VALUES LESS THAN (TO_DAYS('${date}')));`
+ 删除分区：  `ALTER TABLE REGSAPICallAudit DROP PARTITION` \`p2019-06-01\`,\`p2019-06-02\`,\`p2019-06-03\`;
+ 重新划分分区：
```
ALTER TABLE regsapicallaudit
REORGANIZE PARTITION `p2020-04-28` INTO (
PARTITION `p2020-04-26` VALUES LESS THAN (737906),
PARTITION `p2020-04-27` VALUES LESS THAN (737907),
PARTITION `p2020-04-28` VALUES LESS THAN (737908)
);
```
注意重新划分分区时， PARTITION 定义的顺序一定是要递增的。

 #### 重做日志文件
 innodb 存储引擎的数据目录下会有两个名为 ib_logfile0、ib_logfile1 的文件，它们是重做日志文件。当实例或者介质失败时（如断电时），重做日志文件就能派上用场，innodb 存储引擎会使用重做日志文件恢复到失败之前的状态，以此来保证数据的完整性和一致性。下面参数会影响重做日志的属性：
 1. innodb_log_file_size ： 指定每个重做日志文件的大小
 2. innodb_log_files_in_group： 指定日志文件组中重做日志文件的数量，默认为2
 3. innodb_log_group_home_dir： 指定日志文件组所在的目录

重做日志文件与二进制日志文件的区别？ 
1. 二进制日志会记录所有与mysql数据库有关的日志记录，包括 innodb/myisam/heap 等其他存储引擎的日志，而innodb存储引擎的重做日志文件只记录innodb存储引擎本身的事务日志。
2. 二进制日志记录的是一个事务的具体操作内容，是对应的SQL语句，是逻辑日志。而重做日志文件记录的是关于每个页更改的物理情况。
3. 写入时间不同。二进制日志文件仅在事务提交前进行提交，即只写磁盘一次，无论该事务多大。而在事务进行的过程中，却不断有重做日志条目被写入到重做日志文件中。