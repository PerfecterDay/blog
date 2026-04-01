# mysql高可用-主从复制
{docsify-updated}



## 复制原理
复制是 mysql 数据库提供的一种高可用高性能解决方案，分为3个步骤：
1. 主服务器把数据更改（写）记录到二进制日志文件中，
2. 从服务器把主服务器的二进制日志复制到自己的中继日志中（slave IO 线程）
3. 从服务器重做中继日志中的日志，把更改应用到本地数据库上，以达到数据的最终一致性。

<center><img src="pics/mysql-replacation.jpg" width="60%"></center>

### 一般步骤
0. **必须在主从服务器上同时创建好需要同步的数据库及表**，切记
1. 首先在 master 主机服务器上新建用来同步复制 binlog 的账户
   ```
   create user repl@'%' identified by 'replGjgj0831!';
   grant replication slave on *.* to 'repl'@'%';
   flush privileges;
   ```
2. 配置 master 服务器
   ```
   log-bin=master-bin #二进制文件名称
   binlog-format=ROW  #二进制日志格式，有row、statement、mixed三种格式，row指的是把改变的内容复制过去，而不是把命令在从服务器上执行一遍，statement指的是在主服务器上执行的SQL语句，在从服务器上执行同样的语句。MySQL默认采用基于语句的复制，效率比较高。mixed指的是默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。
   server-id=1                #要求各个服务器的id必须不一样
   binlog-do-db=test   #同步的数据库名称
   binlog-ignore-db=mysql # 指定忽略的数据库
   ```
3. 配置slave 服务器
   ```
   server-id=2
   ```
4. 要在 slave 机器设置：
```
mysql> CHANGE MASTER TO MASTER_HOST='master_host_name', //master 地址
MASTER_USER='replication_user_name',  //复制的用户名
MASTER_PASSWORD='replication_password', // 复制的密码
MASTER_LOG_FILE='recorded_log_file_name',  // 复制的日志文件名
MASTER_LOG_POS=recorded_log_position; //指定位置开始

CHANGE MASTER TO MASTER_HOST='10.4.153.131', MASTER_USER='repl', MASTER_PASSWORD='replGjgj0831!';
CHANGE MASTER TO MASTER_HOST='10.4.153.131', MASTER_USER='repl', MASTER_PASSWORD='replGjgj0831!',MASTER_LOG_POS=0;
```
然后，开启 slave 线程：`start slave`.
关闭 slave 线程 : `stop slave`.

使用 `show master status` 或 `show slave status` 可以分别查看主/从服务器上的复制状态。
<center><img src="pics/slave-status.jpg" width="60%"></center>

### Docker主从复制实战
1. 下载 mysql 的docker 镜像：`docker pull mysql`
2. 启动主mysql：`docker run -p 3316:3306 -e MYSQL_ROOT_PASSWORD=123456 -d --name=mysql-master  mysql`
3. 启动从mysql：`docker run -p 3326:3306 -e MYSQL_ROOT_PASSWORD=123456 -d --name=mysql-slave  mysql`
4. 进入docker容器，允许 root 任意IP登录： `update mysql.user set host='%' where user='root';`,然后 `flush privileges`
5. 宿主机连接mysql测试：`mysql -hlocalhost --protocol=TCP -P3316 -umaster -p`，切记要加上 --protocol=TCP 选项
6. 宿主机新建主服务器配置文件 master.cnf:
    ```
	[mysqld]
	#在mysqld模块中添加如下配置信息
	log-bin=master-bin #二进制文件名称
	binlog-format=ROW  #二进制日志格式，有row、statement、mixed三种格式，row指的是把改变的内容复制过去，而不是把命令在从服务器上执行一遍，statement指的是在主服务器上执行的SQL语句，在从服务器上执行同样的语句。MySQL默认采用基于语句的复制，效率比较高。mixed指的是默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。
	server-id=1		   #要求各个服务器的id必须不一样
	binlog-do-db=test   #同步的数据库名称
    binlog-ignore-db=mysql # 指定忽略的数据库
	```
7. 复制 master.cnf 到 mysql-master 的 /etc/mysql/conf.d目录下：  
    `docker cp master.cnf mysql-master:/etc/mysql/conf.d`
8. 宿主机新建主服务器配置文件 slave.cnf:
    ```[mysqld]
    server-id = 2
    # relay_log = relay_bin
    # relay-log-index = relay-bin.index```
9. 复制 slave.cnf 到 mysql-slave 的 /etc/mysql/conf.d目录下：  
    `docker cp slave.cnf mysql-slave:/etc/mysql/conf.d`
10. 重启主从服务器：`docker restart mysql-master mysql-slave`
11. 进入 mysql-master 主容器，创建同步账号并授权： 
    + 创建同步账号：`create user repl@'%' identified by 'repl';`
    + mysql8 可能需要修改密码认证方式：`ALTER USER 'repl'@'%' IDENTIFIED WITH mysql_native_password;`
    + 授权：`GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO repl@'%';`
    + 查看同步状态：`show master status;`
    + 记下容器 IP：`hostname -I` -> 假设为 172.17.0.2
12. 进入 mysql-slave 从容器，配置 slave：
    + 配置同步账号信息：`CHANGE MASTER TO MASTER_HOST='172.17.0.2', MASTER_USER='repl', MASTER_PASSWORD='repl';`
    + 启动同步：`start slave;`
    + 查看同步状态：`show slave status\G;`

## 问题排查
主从复制出错时，使用下述语句查看报错原因：

```sql
select * from performance_schema.replication_applier_status_by_worker;
```

使用下述语句可以跳过报错语句：

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000XXX' LIMIT 200;

STOP SLAVE;
START SLAVE UNTIL MASTER_LOG_FILE='mysql-bin.000XXX', MASTER_LOG_POS=99;

CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000XXX', MASTER_LOG_POS=101;
START SLAVE;

-------------

STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;
```