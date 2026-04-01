# mysql 历史sql查询
{docsify-updated}



## 开启操作记录日志
首先进入mysql输入指令

    show variables like 'gen%';
可以看到输出

    +------------------+-------------------------------------+
    | Variable_name    | Value                               |
    +------------------+-------------------------------------+
    | general_log      | OFF                                 |
    | general_log_file | /usr/local/mysql/data/localhost.log |
    +------------------+-------------------------------------+
可以看到general_log是开启还是环比状态，以及这个帐号的general_log文件在哪，设置开启

    set global general_log=ON;
    commit;//如果关闭了自动提交，记得commit一次结束事务
然后就可以去general_log_file的路径查看操作记录了

## 采用数据库内部查看
除了可以用日志文件的形式查看数据库操作记录之外，也可以把日志存储在一个表，在数据库内部查看

    show variables like '%log_output%';
可以看到输出，

    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | log_output    | FILE  |
    +---------------+-------+
然后将其改为表单

    set  global log_output='TABLE';
之后就可以通过以下两句话查看数据库操作记录

    select * from mysql.general_log; <=====查看操作记录
    truncate table mysql.general_log; <=====清空操作记录表单


