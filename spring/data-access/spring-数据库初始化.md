# 数据库初始化
{docsify-updated}
> https://docs.spring.io/spring-boot/how-to/data-initialization.html

## Spring
```xml
<jdbc:initialize-database data-source="dataSource"
	enabled="#{systemProperties.INITIALIZE_DATABASE}"> 
	<jdbc:script location="..."/>
</jdbc:initialize-database>
```

## Springboot

### 使用 Hibernate 初始化数据库
你可以设置 `spring.jpa.hibernate.ddl-auto` 来控制 Hibernate 的数据库初始化。支持的值有：`none`, `validate`, `update`, `create`, 和 `create-drop` 。Spring Boot 会根据你是否使用嵌入式数据库为你选择一个默认值。hsqldb、h2 或 derby 是嵌入式数据库，而其他数据库则不是。如果识别到嵌入式数据库，且未检测到模式管理器（Flyway 或 Liquibase），`ddl-auto` 默认为创`create-drop`。在所有其他情况下，默认为 `none` 。

从内存数据库切换到 "真实 "数据库时要注意，不要假设新平台中存在表和数据。要么明确设置 `ddl-auto`，要么使用其他机制初始化数据库。

此外，如果 Hibernate 从头开始创建模式（即 `ddl-auto` 属性设置为 `create` 或 `create-drop` ），classpath 根目录下名为 import.sql 的文件会在启动时被执行。如果你小心谨慎，这对演示和测试可能很有用，但不要在生产过环境使用。这是 Hibernate 的一项功能（与 Spring 无关）。

### 使用基本 SQL 脚本初始化数据库
Spring Boot 可以自动创建 JDBC 数据源或 R2DBC ConnectionFactory 的模式（DDL 脚本）并初始化其数据（DML 脚本）。

默认情况下，它会从 `optional:classpath*:schema.sql` 加载 schema 脚本(DDL)，从 `optional:classpath*:data.sql` 加载数据脚本(DML)。这些模式和数据脚本的位置可分别使用 `spring.sql.init.schema-locations` 和 `spring.sql.init.data-locations` 进行自定义。`optional:`前缀表示即使文件不存在，应用程序也会启动。要使应用程序在文件不存在时无法启动，请移除 `optional:` 前缀。

默认情况下，只有在使用嵌入式内存数据库时才会执行 SQL 数据库初始化。要始终初始化 SQL 数据库（无论其类型如何），请将 `spring.sql.init.mode` 设为 `always` 。同样，要禁用初始化，请将 `spring.sql.init.mode` 设为 `never` 。默认情况下，Spring Boot 会启用基于脚本的数据库初始化程序的 `fail-fast` 功能。这意味着，如果脚本导致异常，应用程序将无法启动。可以通过设置 `spring.sql.init.continue-on-error` 来调整该行为。

基于脚本的数据源初始化默认会在创建任何 JPA `EntityManagerFactory` Bean 之前执行。`schema.sql` 可用于为 JPA 管理的实体创建 schema ，而 `data.sql` 可用于初始化数据库数据。虽然我们不建议使用多种数据源初始化技术，但如果希望基于脚本的数据源初始化能在基于 Hibernate 初始化之后执行，可将 `spring.jpa.defer-datasource-initialization` 设为 true。这样，数据源初始化将推迟到任何 `EntityManagerFactory` Bean 创建并初始化之后。然后，`schema.sql` 可用于对 Hibernate 初始化的数据库进行填充，data.sql 可用于对其进行填充。

```yml
spring:
  sql:
    init:
      continue-on-error: true
      mode: always
      schema-locations: classpath*:db-init/schema.sql
```

## [使用高级工具初始化](https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.migration-tool)