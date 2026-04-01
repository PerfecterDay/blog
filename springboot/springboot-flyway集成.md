# Springboot 集成 flyway
{docsify-updated}

> https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.migration-tool

mysql 从 8.0.31 开始，artifactId 已统一为 `mysql-connector-j` ，不是以前的 `mysql-connector-java` 。
```
<dependency>
  <groupId>com.mysql</groupId>
  <artifactId>mysql-connector-j</artifactId>
  <version>9.0.0</version>
</dependency>
```

Spring Boot 4.0.x 中， `Flyway` 已经不再由 `spring-boot-autoconfigure` 提供自动配置了，从 Spring Boot 4.0 开始，大量可选第三方集成被模块化拆分，数据库迁移工具（Flyway / Liquibase）被移出 `core autoconfigure` 。 所以需要手动添加依赖：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-flyway</artifactId>
</dependency>

<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

`application.yml` 配置:
```
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/cap
    username: root
    password: 123456

  flyway:
    enabled: true
    locations: classpath:db/migration
```

在 `resources` 文件夹下创建以下文件
```

└── db
    └── migration
        └── V1__create_user_table.sql
```

注意，启动项目之前必须先创建好数据库。

## 文件协议
1. 版本迁移（ Versioned Migration ）
```
V<主版本>_<子版本>__<动词>_<对象>.sql
```
flyway 只会执行一次，一旦确定了一个版本文件，执行后就不要

2. 可重复迁移 ( Repeatable Migration )
```
R__<描述>.sql
```
没有版本号，只要内容有变动，就会重新执行，执行顺序是在所有 V 之后。  
主要是用来创建view、procedure、function以及执行数据初始化操作。

3. 目录规范
```
db/migration
├── V1_0__init_schema.sql
├── V1_1__create_user_table.sql
├── V1_2__add_user_index.sql
├── R__create_view_user_summary.sql
```


## 问题
1. mysql 版本过高，返回不支持Mysql 9.5 ,异常栈
```
Caused by: org.flywaydb.core.api.FlywayException: Unsupported Database: MySQL 9.5
	at org.flywaydb.core.internal.database.DatabaseTypeRegister.lambda$getDatabaseTypeForConnection$7(DatabaseTypeRegister.java:142) ~[flyway-core-11.14.1.jar:na]
	at java.base/java.util.Optional.orElseThrow(Optional.java:403) ~[na:na]
	at org.flywaydb.core.internal.database.DatabaseTypeRegister.getDatabaseTypeForConnection(DatabaseTypeRegister.java:142) ~[flyway-core-11.14.1.jar:na]
	at org.flywaydb.core.internal.jdbc.JdbcConnectionFactory.<init>(JdbcConnectionFactory.java:77) ~[flyway-core-11.14.1.jar:na]
    ....
```

解决方案，添加以下依赖告知 flyway 使用的 mysql ，否则默认只有 H2/SqlLite/TestContainers 的支持：
```
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

## 修复数据库
添加 flyway 插件:
```
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <dependencies>
        <!-- MySQL 支持 -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-mysql</artifactId>
            <version>${flyway.version}</version>
        </dependency>

        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>${mysql.version}</version>
        </dependency>
    </dependencies>
</plugin>
```

```
mvnd flyway:repair \
  -Dflyway.url=jdbc:mysql://localhost:3306/cap \
  -Dflyway.user=root \
  -Dflyway.password=root
```

repair 的本质就是：修复 flyway_schema_history 表
1. 删除失败的 migration 记录
2. 重新计算 checksum（最关键）
3. 修复 metadata（描述信息）


❌ 不会执行 SQL
❌ 不会修改表结构
❌ 不会回滚数据
❌ 不会重新跑 migration

repair = 修改 Flyway 的“记账本”，不是修改数据库