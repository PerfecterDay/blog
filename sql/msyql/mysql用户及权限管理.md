# mysql 用户及权限管理
{docsify-updated}

## 用户管理
mysql 用户信息存储在 mysql.user 表中。
1. 创建用户： `CREATE USER 'username'@'ip' IDENTIFIED WITH mysql_native_password BY 'passowrd'`;
2. 允许用户从任意IP登录：`update mysql.user set host='%' where user='root';`
3. 修改用户密码： `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的密码';` mysql 8  
`update mysq.user set password=password('11111111') where xxx;` 8 以前的版本
3. 删除用户： `drop user testuser@'localhost'`
4. 刷新用户权限：`flush privileges`
5. 查看当前登录的用户：`select user();` 或 `SELECT CURRENT_USER();`

## 授权与收回
SQL标准包括 select、insert、update、delete以及all权限。

0. 查看用户的权限：`show grants for <user>@<host>`
1. 授权语句：
    ```
    grant <权限列表>
    on <数据库/表/视图>
    to <用户/角色列表>

	grant create,select,insert,update on user.* to 'app'@'%';
    grant select on department to public,Amy,Simith;授予public,Amy,Simith 关于 department 的select 权限  
    grant all on *.* to admin; 授予admin所有权限  
    grant all on student.* to teacher;授予teacher所有student数据库相关的权限
    ```
    public代表系统中所有的当前用户和将来的用户。

    ```
    GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EVENT, TRIGGER ON *.* TO `testuser`@`%` WITH GRANT OPTION
    ```
    `WITH GRANT OPTION` 代表是使得该用户可以拥有权限和回收权限给其他用户。

2. 收回权限语句：
    ```
    revoke <权限列表>
    on <关系名或视图名>
    from <用户/角色列表>

    revoke select on department from Amy,Simith
	revoke select,insert,update,create on user.user from 'app'@'%';

    revoke grant option on *.* from testuser@'%';
    ```
