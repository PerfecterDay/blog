# @Transactional注解
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html


除了基于 XML 的声明式事务配置方法外，还可以采用基于注解的方法。直接在 Java 源代码中声明事务语义，能使声明与受影响的代码更加贴近。这种做法几乎不会导致不必要的耦合，因为本就应以事务方式使用的代码，通常也会以这种方式部署。

直接看示例：
```
@Transactional
public class DefaultFooService implements FooService {
	@Override
	public Foo getFoo(String fooName) {
		// ...
	}

	@Override
	public void insertFoo(Foo foo) {
		// ...
	}

	@Override
	public void updateFoo(Foo foo) {
		// ...
	}
}
```
如上所述，在类级别使用该注解时，注解表示声明类所有方法（默认是 `public/protect` 方法）都是事务管理的。`@Transactional` 也可以单独注解在某个个方法上。请注意，**`@Transactional` 注解不会被继承，如果一个子类继承了一个 `@Transactional` 注解的父类，需要在子类或者其方法中重新加上 `@Transactional` 才能使事务生效。**

仅有 `@Transactional` 注解还不足以激活事务行为。 `@Transactional` 注解只是一个元数据，要使 `@Transactional`注解生效，还要在一个 `@Configuration` 配置类中加上 `@EnableTransactionManagement` 。或者在 xml 中使用 `<tx:annotation-driven/>` 。

Spring 团队建议您在具体类的的方法上添加 @Transactional 注解，而不是依赖接口中的注解方法——即使在 5.0 版本中，后者对于基于接口和目标类的代理确实有效。由于 Java 注解不会从接口继承，因此在使用 AspectJ 模式时，接口声明的注解仍无法被编织基础设施识别，从而导致切面无法应用。因此，事务注解可能会被静默忽略：代码看似“正常运行”，直到测试回滚场景时才会发现问题。

`@Transactional` 注解通常用于可见性为 `public` 的方法。从 6.0 版本开始，对于基于类的代理，受保护（ `protected` ）或包可见（ `package-visible` ）的方法默认也可设置为事务性方法。请注意，在基于接口的代理中，事务性方法必须始终为 `public` 且定义在被代理的接口中。对于这两种代理，只有通过代理传入的外部方法调用才会被拦截。

## @Transactional的属性设置
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    String[] label() default {};

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    String timeoutString() default "";

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```

<center><img src="pics/transactional.png" alt=""></center>

`@Transactional` 注解是一种元数据，用于指定接口、类或方法的具有事务语义。默认的 `@Transactional` 设置如下：
+ 传播特性设置为 `PROPAGATION_REQUIRED` 
+ 隔离级别为 `ISOLATION_DEFAULT` 
+ `readOnly` 为 `false`
+ 事务超时默认为底层事务系统的默认超时，如果不支持超时，则默认为无。
+ 任何 `RuntimeException` 或 `Error` 都会触发回滚，而任何 `checked Exception` 则不会触发回滚。

从 6.2 版本开始，可以全局更改默认回滚行为——例如，通过 `@EnableTransactionManagement(rollbackOn=ALL_EXCEPTIONS)` 注解，这将导致事务内抛出的所有异常（包括任何受检查异常）都会触发回滚。若需进一步自定义， `AnnotationTransactionAttributeSource` 提供了 `addDefaultRollbackRule(RollbackRuleAttribute)` 方法，用于定义自定义的默认规则。

请注意，事务特定的回滚规则会覆盖默认行为，但对于未指定的异常仍保留所选的默认行为。Spring 的 `@Transactional` 注解以及 JTA 的 `jakarta.transaction.Transactional` 注解均遵循此原则。

除非依赖具有提交行为的 EJB 风格业务异常，否则建议切换到 `ALL_EXCEPTIONS` ，以确保即使在发生（可能是意外的）受检查异常时，回滚语义也能保持一致。此外，对于完全不强制检查受检查异常的 Kotlin 应用程序，也建议进行此切换。

目前，无法显式控制事务的名称，这里的“名称”指的是在事务监视器和日志输出中显示的事务名称。对于声明式事务，事务名称始终是**事务增强类的完全限定类名 + . + 方法名**。例如，如果 `BusinessService` 类的 `handlePayment(..)` 方法启动了一个事务，则该事务的名称将为 `com.example.BusinessService.handlePayment` 。

## 指定多个事务管理器
大多数 Spring 应用程序只需一个事务管理器，但在某些情况下，可能希望在一个应用程序中使用多个独立的事务管理器。可以使用 `@Transactional` 注解的 `value` 或 `transactionManager` 属性，可选地指定要使用的 `TransactionManager` 的标识。该标识可以是 Bean 名称，也可以是事务管理器 Bean 的 `qualifier` 值。例如，使用 `qualifier` 表示法，可以将以下 Java 代码与应用程序上下文中以下事务管理器 Bean 声明结合使用：
```
public class TransactionalService {

	@Transactional("order")
	public void setSomething(String name) { ... }

	@Transactional("account")
	public void doSomething() { ... }

	@Transactional("reactive-account")
	public Mono<Void> doSomethingReactive() { ... }
}

<bean id="transactionManager1" class="org.springframework.jdbc.support.JdbcTransactionManager">
	...
	<qualifier value="order"/>
</bean>

<bean id="transactionManager2" class="org.springframework.jdbc.support.JdbcTransactionManager">
	...
	<qualifier value="account"/>
</bean>

<bean id="transactionManager3" class="org.springframework.data.r2dbc.connection.R2dbcTransactionManager">
	...
	<qualifier value="reactive-account"/>
</bean>
```

## 自定义注解
```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional(transactionManager = "order", label = "causal-consistency")
public @interface OrderTx {
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional(transactionManager = "account", label = "retryable")
public @interface AccountTx {
}
```

## 事务失效
因为事务原理是基于 AOP 代理的，所以只有通过代理引用进入的外部方法调用才会被拦截。这意味着自调用（实际上是目标对象中的一个方法调用目标对象的另一个方法）在运行时不会导致实际事务，即使被调用的方法标记了 `@Transactional` 也是如此。此外，代理必须完全初始化才能正常工作，因此不应在初始化代码中依赖这一功能，例如在 `@PostConstruct` 方法中期望使用spring 事务管理功能。

## 基于XML的声明式事务实例
```xml
<bean id="fooService" class="x.y.service.DefaultFooService"/>

<!-- the transactional advice (what 'happens'; see the <aop:advisor/> bean below) -->
<tx:advice id="txAdvice" transaction-manager="txManager">
	<!-- the transactional semantics... -->
	<tx:attributes>
		<!-- all methods starting with 'get' are read-only -->
		<tx:method name="get*" read-only="true"/>
		<!-- other methods use the default transaction settings (see below) -->
		<tx:method name="*"/>
	</tx:attributes>
</tx:advice>

<!-- ensure that the above transactional advice runs for any execution
	of an operation defined by the FooService interface -->
<aop:config>
	<aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
	<aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
</aop:config>

<!-- don't forget the DataSource -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
	<property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
	<property name="username" value="scott"/>
	<property name="password" value="tiger"/>
</bean>

<!-- similarly, don't forget the TransactionManager -->
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
</bean>
```
该配置用于在根据 `fooService` Bean 定义创建的对象周围创建事务代理proxy。代理通过事务 advice 进行增强配置，这样，当在代理上调用适当的方法时，就会根据与该方法相关的事务配置，启动、暂停或标记为只读等事务。

## 基于XML的事务回滚
在默认配置中，Spring Framework 的事务基础架构代码仅在 `runtime`, `unchecked exceptions` 异常时才会标记事务以进行回滚。也就是说，当抛出的异常是 `RuntimeException` 或者 `Error` 的实例或子类时，会导致事务回滚。

在默认配置中，事务方法抛出的 `checked exceptions` 不会导致回滚。不过可以通过指定回滚规则，准确配置哪些异常类型（包括`checked exceptions`）会导致事务回滚。
```
<tx:advice id="txAdvice" transaction-manager="txManager">
	<tx:attributes>
		<tx:method name="get*" read-only="true" rollback-for="NoProductInStockException"/>
		<tx:method name="updateStock" no-rollback-for="InstrumentNotFoundException"/>
		<tx:method name="*"/>
	</tx:attributes>
</tx:advice>
```