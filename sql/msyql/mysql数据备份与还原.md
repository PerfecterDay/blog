#  mysql数据备份与还原
{docsify-updated}


### 备份概述
根据备份方法不同，可以分为三类：
1. 热备，数据库不停止，备份不会对数据库产生影响
2. 冷备，数据库停止，复制相关文件即可
3. 温备，数据库不停止，但是备份会影响数据库操作，如加一个全局读锁以保证备份数据的一致性

根据备份后文件的内容可以分为：
1. 裸文件备份
2. 逻辑备份

裸文件备份是指备份数据库的物理文件，可以使用相关工具不停机复制，也可以停机复制。恢复时间短，但是备份内容不可读。

逻辑备份是指备份出的文件内容是可读的，一般是文本文件。内容是指由一条条 sql 语句或者表数据构成。优点是可以查看到处文件的内容，但是恢复时间较长。

数据可备份的一致性：在备份的时间点上，数据库中的数据是一致的。假设，正在备份时，网络游戏中某个玩家在购买道具，先扣除相应金钱，然后发放装备。必须确保备份后的数据是一致的。不能扣钱不发装备或者发了装备不扣钱。可以开启一个事务，然后备份相关表，最后提交。扣钱-发装备操作必须在一个事务中完成。

### 冷备
对于 Innodb 引擎很简单，只需要备份 Mysql 数据库的 frm 文件（mysql8删除了这个文件，将表定义放在在ibd文件中）、共享表空间文件、独立表空间文件(*.ibd)、重做日志文件。优点是：
1. 备份简单，只需要复制相关文件即可
2. 恢复简单，将备份文件复制到对应目录即可
3. 恢复的速度快，不需要执行sql语句，也不用重建索引
4. 跨操作系统，跨版本

缺点：
1. 备份文件比较大，除了数据文件外，包含了数据库中其它数据。
2. 也不总是可以跨平台。操作系统、数据库版本、浮点数格式、文件大小写敏感都会成为问题。

### 逻辑备份

#### mysqldump 备份
`mysqldump [arguments] > file_name`，不能导出视图。

```
mysqldump --databases nrng -uroot -p > nrng.sql
rsync ./nrng.sql 10.4.153.130:/root/mysql
mysqldump -uroot -p nrng < nrng.sql


mysqldump -u root -p block_chain_management system_users --no-create-info --skip-triggers --skip-add-locks --skip-comments > data.sql
mysqldump -u root -p block_chain_management system_users --no-create-info --skip-triggers --skip-add-locks --skip-comments --skip-extended-insert > data.sql

mysqldump --opt -d cap -u appadmin -p > create.sql
```

重要参数：

0. `--all-databases`: 备份所有数据库。
1. `--no-create-info`: 不导出建表语句，只导出数据
2. `--databases db_name1 db_name1 db_name1`: 备份指定数据库
3. `--single-transaction` : 在备份开始时，先执行 `start transaction` 语句开启事务，保证数据一致性，但是不能执行DDL语句，不能隔离DDL操作。且只对 Innodb 引擎有效
4. `--lock-tables(-l)`: 备份时，依次锁住数据库中每个架构下的所有表来保证单表的一致性，不能保证整个数据库的一致性。一般用于 MyIsam 引擎，且与 `--single-transaction` 互斥，不能同时使用。这个选项默认是打开的，也就是说备份时默认是锁表的，如果是热备，那么线上只能执行读语句，所有写事务都会被阻塞。
5. `--lock-all-tables(-x)`: 备份时，锁住所有架构下的所有表， 避免 --lock-tables 不能锁住所有表的问题。
6. `--add-drop-tables`:在 create database 前先运行 drop database。此参数要和 --al-databases 或者 --databases 选项一起使用。
7. `--master-data[=value]`： 通过改参数产生的备份文件主要用来建立一个 replication。value为1时，文件中记录 change master 语句；为2时，change master 语句被写出 sql注释。
8. `--events(-E)`: 备份事件调度器
9. `--routines(-R)`: 备份存储过程和函数。
10. `--triggers`：备份触发器
11. `--hex-blob`：将 binary、varbinary、blob、bit类型的数据备份为十六进制的格式，默认是不可读的乱码。
12. `--where='where_condition'(-w 'where_condition')`：导出给定条件的数据。
13. `--opt`：只导出表结构

#### mysqldump 恢复
1. `mysql -uroot -p < file_name`
2. 登录到mysql后，执行：`source file_name`

#### SELECT...INTO OUTFILE 备份
```
select * from Table into outfile '/路径/文件名'
fields terminated by ','
enclosed by '"'
lines terminated by '\r\n'
```
导出一张表中的数据到文件.

#### SELECT...INTO OUTFILE 恢复
1. load data infile    
2. mysqlimport [options] db_file

#### 二进制日志备份与恢复
首先需要开启二进制日志功能：
    log-bin=mysql-bin
    sync_binlog=1
    innodb_support_xa=1
备份二进制文件前，使用 `flush logs` 命令生成一个新的二进制日志文件，然后备份之前的二进制文件。

恢复二进制日志：
`mysqlbinlog [options] bin_log_file | mysql -u root -p`
--start-position 和 --stop-position 可以用来指定从二进制日志的某个偏移量进行恢复。


#### [Mysql 数据恢复](https://www.stellarinfo.com/blog/repair-innodb-table-corruption-in-mysql/)

zentao 启动脚本(/.docker_init.sh)：
```
if [ "`ls -A /var/lib/mysql/`" = "" ]; then
  mysql_install_db --defaults-file=/etc/mysql/my.cnf
  /etc/init.d/mysql restart
  /usr/bin/mysqladmin -uroot password $MYSQL_ROOT_PASSWORD
else
  /etc/init.d/mysql restart -p 123456
fi
```

+ 主机的mysql数据文件：/Users/demo/www/mysqldata/zentao:var/lib/mysql
+ mysql日志： /var/log/mysql/error.log

配置文件中(/etc/mysql/mariadb.conf.d/50-server.cnf)增加下述配置：
```
[mysqld]
innodb_force_recovery=1
```

dump 数据：
```
mysqldump -u user -p database_name table_name > single_dbtable_dump.sql
mysqldump -u root -p zentao zt_bug > zt_bug_dump.sql
```

删除老的数据：
```
mysql -u user -p –execute=”DROP TABLE database_name.table_name”
mysql -u root -p –execute=”DROP TABLE zentao.zt_bug
```

还原旧的数据：
```
mysql -u user -p < single_dbtable_dump.sql
mysql -u root -p < zt_bug_dump.sql
```
将配置还原：
```
[mysqld]
#innodb_force_recovery=1
```