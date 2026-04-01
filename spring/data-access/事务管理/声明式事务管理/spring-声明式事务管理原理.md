# 声明式事务原理
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html

仅仅告诉大家在类上添加 `@Transactional` 注解、在配置中加入 `@EnableTransactionManagement` ，并期望大家能理解其工作原理，这是远远不够的。为了帮助大家更深入地理解，本节将结合事务相关的问题，详细讲解 Spring 框架声明式事务基础设施的内部工作原理。

关于 Spring Framework 的声明式事务支持，需要掌握的最重要的概念是：这种支持是通过 AOP 代理实现的，事务相关的 `advice` 是由元数据（目前基于 XML 或注解）驱动的。AOP 与事务元数据的结合产生了一个 AOP 代理，它使用事务拦截器 `TransactionInterceptor` 与适当的事务管理器 `TransactionManager` 实现来驱动方法调用的增强，从而实现事务管理功能。

Spring Framework 的 `TransactionInterceptor` 为命令式和反应式编程模型提供事务管理。`TransactionInterceptor` 通过检查方法的返回类型来检测所需的事务管理类型。返回 `Publisher` 或 Kotlin `Flow`（或其子类型）等反应式类型的方法使用反应式事务管理 `ReactiveTransactionManager` 。包括 `void` 在内的所有其他返回类型都使用命令式事务管理 `PlatformTransactionManager` 。

<center><img src="pics/tx.png" alt=""></center>

## Springboot 配置相关类
```
ProxyTransactionManagementConfiguration
```