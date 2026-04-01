#  mysql的安装及配置
{docsify-updated}

- [mysql的安装及配置](#mysql的安装及配置)
    - [Linux Redhat 8 下安装 mysql](#linux-redhat-8-下安装-mysql)
    - [windows-mysql 安装](#windows-mysql-安装)

### Linux Redhat 8 下安装 mysql
Mysql官网下载以下RPM包：
```
mysql-community-client-8.0.33-1.el8.x86_64.rpm
mysql-community-client-plugins-8.0.33-1.el8.x86_64.rpm
mysql-community-common-8.0.33-1.el8.x86_64.rpm
mysql-community-icu-data-files-8.0.33-1.el8.x86_64.rpm
mysql-community-libs-8.0.33-1.el8.x86_64.rpm
mysql-community-server-8.0.33-1.el8.x86_64.rpm
```
执行 `yum localinstall [rpm包]` 命令，提示缺少啥依赖包，就先装依赖。一般顺序如下：
```
yum localinstall mysql-community-common-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-client-plugins-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-libs-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-client-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-icu-data-files-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-server-8.0.33-1.el8.x86_64.rpm -y
```


如果只需要安装客户端：
```
yum localinstall mysql-community-common-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-client-plugins-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-libs-8.0.33-1.el8.x86_64.rpm -y &&\
yum localinstall mysql-community-client-8.0.33-1.el8.x86_64.rpm -y
```


+ `systemctl start mysqld` 启动服务。
+ `cat /var/log/mysqld.log | grep temporary`: 查询首次安装的登录账号和密码
+ 在配置文件 `/ect/my.cnf` 的 mysqld 中加上 `skip-grant-tables` 配置，然后启动 `systemctl start mysqld`，使用 mysql 可以直接登录.
    然后清空root密码：`UPDATE mysql.user SET authentication_string='' WHERE user='root';`
    接着就可以无密码登录了，登录后直接运行`ALTER USER 'root'@'localhost' IDENTIFIED BY 'Jhqqt0711!';`修改密码
+ `ALTER USER 'root'@'localhost' IDENTIFIED BY 'Jhqqt0711!';` //修改密码
+ `update mysql.user set host='%' where user='root';` //修改允许登录的地址
+ `CREATE USER 'app'@'%' IDENTIFIED BY 'Jhqqt0711!'` //创建一个app用户

### windows-mysql 安装
1. 下载zip包，解压到某个目录
2. 解压目录下创建my.ini文件，配置basedir和datadir路径
   ```
    [mysqld]
    # set basedir to your installation path
    basedir=C:/Program Files (x86)/mysql-8.0.21-winx64
    # set datadir to the location of your data directory
    datadir=D:/mysqldata/data
    ```
3. 执行 mysqld --initialize --console --user=root ,初始化数据库，会在控制台打印出root账户的随机密码。必须保证配置的 data 目录是空的，否则会报错。
4. 启动mysqld服务：mysqld
5. 停止mysqld服务：`mysqladmin -u root shutdown`，如果要密码加上-p
6. 执行mysql -hlocalhost -uroot -p连接数据库
7. 首次连接数据库后要更改初始化密码：ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';否则不能执行sql语句
8. 允许 root 从任意IP登录：`update mysql.user set host='%' where user='root';`,然后 `flush privileges`

### windows-mysql 安装为服务
1. `mysqld --install-manual MySQL --defaults-file="D:\applications\Scoop\apps\mysql\current\my.ini"` 安装为手动启动的服务，`mysqld --install MySQL --defaults-file="D:\applications\Scoop\apps\mysql\current\my.ini"` 安装位自动启动的服务
2. 启动：`sc start mysql`，停止：`sc stop mysql`
2. 卸载服务:`sc delete MySQL`

### 常见的 mysql 命令行工具
```
$ tree /opt/homebrew/opt/mysql/bin/
├── comp_err
├── ibd2sdi
├── innochecksum
├── my_print_defaults
├── myisam_ftdump
├── myisamchk
├── myisamlog
├── myisampack
├── mysql
├── mysql.server -> ../support-files/mysql.server
├── mysql_client_test
├── mysql_config
├── mysql_config_editor
├── mysql_keyring_encryption_test
├── mysql_migrate_keyring
├── mysql_secure_installation
├── mysql_test_event_tracking
├── mysql_tzinfo_to_sql
├── mysqladmin
├── mysqlbinlog
├── mysqlcheck
├── mysqld
├── mysqld_multi
├── mysqld_safe
├── mysqldump
├── mysqldumpslow
├── mysqlimport
├── mysqlrouter
├── mysqlrouter_keyring
├── mysqlrouter_passwd
├── mysqlrouter_plugin_info
├── mysqlshow
├── mysqlslap
├── mysqltest
├── mysqltest_safe_process
├── mysqlxtest
└── perror
```

1. `mysqld` 这个可执行文件就代表着 MySQL 服务器程序，运行这个可执行文件就可以直接启动一个服务器进程。但这个命令不常用。
2. `mysqld_safe` 是一个启动脚本，它会间接的调用 `mysqld` ，而且还顺便启动了另外一个监控进程，这个监控进程在服务器进程挂了的时候，可以帮助重启它。另外，使用 mysqld_safe 启动服务器程序时，它会将服务器程序的出错信息和其他诊断信息重定向到某个文件中，产生出错日志，这样可以方便我们找出发生错误的原因。
3. `mysql.server` 也是一个启动脚本，它会间接的调用 `mysqld_safe` ，在调用 `mysql.server` 时在后边指定 `start` 参数就可以启动服务器程序了，就像这样 : `mysql.server start`, 也可以使用 `mysql.server` 命令来关闭正在运行的服务器程序: `mysql.server stop`
4. 一台计算机上也可以运行多个服务器实例，也就是运行多个 MySQL 服务器进程。 `mysql_multi` 可执行文件可以对每一个服务器进程的启动或停止进行监控。


## 问题
1. `ERROR 1524 (HY000): Plugin 'mysql_native_password' is not loaded` 导致mysql 客户端无法连接到服务器
首先，重启mysql 加伤 `--skip-grant-tables` 启动参数， 它是一个非常实用的MySQL启动参数，它允许数据库在启动时不加载授权表。 这意味着在启动过程中，MySQL不会检查用户的权限，所有连接到数据库的用户都将拥有最高权限。 这一特性在某些特定情况下非常有用，尤其是在管理员忘记了自己的管理员密码时。
```
vim /opt/homebrew/etc/my.cnf

[mysqld]
skip-grant-tables
```
或者 
`/opt/homebrew/opt/mysql/bin/mysqld_safe --datadir\=/opt/homebrew/var/mysql --skip-grant-tables &` 启动 mysql
然后连接mysql : `mysql -uroot`.

```
SELECT User, Host, plugin FROM mysql.user WHERE plugin = 'mysql_native_password';
ALTER USER 'root'@'%' IDENTIFIED WITH caching_sha2_password BY 'root';
```
