# Spring 事务管理的核心抽象
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/data-access/transaction/strategies.html

Spring 事务抽象的关键在于事务策略的概念。事务策略由事务管理器（ `TransactionManager` ）定义，特别是用于命令式事务管理的 `org.springframework.transaction.PlatformTransactionManager` 接口和用于反应式事务管理的 `org.springframework.transaction.ReactiveTransactionManager` 接口。以下列表显示了 `PlatformTransactionManager/ReactiveTransactionManager` API 的定义：
```java
public interface PlatformTransactionManager extends TransactionManager {
	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}

public interface ReactiveTransactionManager extends TransactionManager {
	Mono<ReactiveTransaction> getReactiveTransaction(TransactionDefinition definition) throws TransactionException;
	Mono<Void> commit(ReactiveTransaction status) throws TransactionException;
	Mono<Void> rollback(ReactiveTransaction status) throws TransactionException;
}
```
`TransactionManager` 及其子接口主要基于 SPI 模式工作，也就是说各个服务供应商通过 SPI 机制提供其实现，不过也可以在应用程序代码中以编程方式使用它。由于 `PlatformTransactionManager` 是一个接口，因此可以根据需要轻松地对其进行模拟或存根处理。它与查找策略（如 JNDI）无关。 `PlatformTransactionManager` 的实现与 Spring Framework IoC 容器中的其他对象（或 Bean）一样定义。仅这一点就足以让 Spring Framework 事务成为一个值得使用的抽象，即使在使用 JTA 时也是如此。与直接使用 JTA 相比，可以更轻松地测试事务代码。

同样，遵循 Spring 的设计理念， `PlatformTransactionManager` 接口中任何方法都可能抛出的 `TransactionException` 属于未检查异常（即，它继承自 `java.lang.RuntimeException` 类）。事务基础设施的故障几乎总是致命的。在极少数情况下，如果应用程序代码确实能够从事务故障中恢复，应用程序开发人员仍可选择捕获并处理 `TransactionException` 。关键在于，开发人员并非必须这样做。

## TransactionDefinition
```
public interface TransactionDefinition {

	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;


	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = 1;  // same as java.sql.Connection.TRANSACTION_READ_UNCOMMITTED;
	int ISOLATION_READ_COMMITTED = 2;  // same as java.sql.Connection.TRANSACTION_READ_COMMITTED;
	int ISOLATION_REPEATABLE_READ = 4;  // same as java.sql.Connection.TRANSACTION_REPEATABLE_READ;
	int ISOLATION_SERIALIZABLE = 8;  // same as java.sql.Connection.TRANSACTION_SERIALIZABLE;


	/**
	 * Use the default timeout of the underlying transaction system,
	 * or none if timeouts are not supported.
	 */
	int TIMEOUT_DEFAULT = -1;


	default int getPropagationBehavior() {
		return PROPAGATION_REQUIRED;
	}


	default int getIsolationLevel() {
		return ISOLATION_DEFAULT;
	}

	
	default int getTimeout() {
		return TIMEOUT_DEFAULT;
	}

	
	default boolean isReadOnly() {
		return false;
	}


	@Nullable
	default String getName() {
		return null;
	}

	static TransactionDefinition withDefaults() {
		return StaticTransactionDefinition.INSTANCE;
	}

}
```

`TransactionDefinition` 接口定义了：
+ 传播机制：就是当在一个方法中调用了多个涉及事务管理的方法时，如何处理事务上下文。通常，事务范围内的所有代码都在**同一个事务**中运行。不过也可以指定当事务上下文已经存在时运行事务方法的行为。例如，代码可以继续在现有事务中运行（常见情况），也可以暂停现有事务并创建新事务。Spring 提供 EJB CMT 中熟悉的所有事务传播选项。要了解 Spring 中事务传播的语义，请参阅事务传播。
+ [隔离级别](/sql/数据库事务.md#事务的隔离级别)：该事务与其他事务工作隔离的程度。
+ 超时：该事务运行多长时间后会超时并被底层事务基础架构自动回滚。
+ 只读状态：当代码读取但不修改数据时，可以使用只读事务。在某些情况下，只读事务是一种非常有用的优化方式，例如在使用 Hibernate 时。

这些设置反映了标准事务概念。如有必要，请参考讨论事务隔离级别和其他核心事务概念的资源。了解这些概念对于使用 Spring 框架或任何事务管理解决方案都至关重要。

### 事务传播机制
事务传播行为在 Spring 中由枚举 `Propagation` 定义：
```
public enum Propagation {
	REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
	SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),
	MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),
	REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
	NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),
	NEVER(TransactionDefinition.PROPAGATION_NEVER),
	NESTED(TransactionDefinition.PROPAGATION_NESTED);

	private final int value;

	Propagation(int value) {
		this.value = value;
	}

	public int value() {
		return this.value;
	}
}
```

<center><img src="pics/propagation.png" width="80%"></center>

#### PROPAGATION_REQUIRED
<center><img src="pics/tx_prop_required.png" width="40%"></center>

当传播设置为 `PROPAGATION_REQUIRED` 时，将为应用该设置的每个方法创建一个逻辑事务作用域。每个逻辑事务作用域都可以单独确定只回滚状态，外部事务作用域在逻辑上独立于内部事务作用域。在标准 `PROPAGATION_REQUIRED` 行为中，所有这些作用域都映射到同一个物理事务。因此，在内部事务作用域中设置的只回滚标记确实会影响外部事务实际提交的机会。

但是，在内部事务作用域设置了只回滚标记的情况下，外部事务本身并没有决定回滚，因此回滚（由内部事务作用域悄悄触发）是意外的。此时会抛出相应的 `UnexpectedRollbackException` 。这是我们所期望的行为，这样事务调用者就不会被误导，以为已经执行了提交，而实际上并没有。因此，如果内部事务（外部调用者不知道）默默地将事务标记为仅回滚，外部调用者仍会调用提交。外层调用者需要收到一个 `UnexpectedRollbackException` 异常，以明确表示执行了回滚。

#### PROPAGATION_REQUIRES_NEW
<center><img src="pics/tx_prop_requires_new.png" width="40%"></center>

与 `PROPAGATION_REQUIRED` 相反， `PROPAGATION_REQUIRES_NEW` 总是为每个受影响的事务范围使用独立的物理事务，从不参与外层范围的现有事务。在这种安排下，底层资源事务是不同的，因此可以独立提交或回滚，外部事务不受内部事务回滚状态的影响，内部事务的锁在完成后立即释放。这种独立的内部事务还可以声明自己的隔离级别、超时和只读设置，而不会继承外部事务的特性。

当内部事务获取自己的资源（如新的数据库连接）时，连接到外部事务的资源将保持绑定。这可能会导致连接池耗尽，如果多个线程有一个活动的外部事务，并等待为其内部事务获取一个新连接，而连接池无法再分配任何此类内部连接，则可能导致死锁。请勿使用 `PROPAGATION_REQUIRES_NEW` ，除非您的连接池大小合适，超过并发线程数至少 1 个。

#### PROPAGATION_NESTED
PROPAGATION_NESTED 使用的是单个物理事务，它可以回滚到多个保存点。这种部分回滚允许内部事务作用域触发其作用域的回滚，尽管某些操作已经回滚，外部事务仍能继续物理事务。此设置通常映射到 JDBC 保存点，因此只适用于 JDBC 资源事务。

## TransactionStatus
`TransactionStatus` 接口为事务代码提供了一种控制事务执行和查询事务状态的简单方法。这些概念应该很熟悉，因为它们是所有事务 API 的共同点。下面的列表显示了 TransactionStatus 接口：
```
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
	@Override
	boolean isNewTransaction();

	boolean hasSavepoint();

	@Override
	void setRollbackOnly();

	@Override
	boolean isRollbackOnly();

	void flush();

	@Override
	boolean isCompleted();
}
```

## 定义 TransactionManager
无论在 Spring 中选择声明式还是编程式事务管理，定义正确的 `TransactionManager` 实现都是绝对必要的。您通常通过依赖注入来定义该实现。 `TransactionManager` 实现通常需要了解其工作环境：JDBC、JTA、Hibernate 等。以下示例展示了如何定义本地 `PlatformTransactionManager` 实现（在本例中，使用的是普通 JDBC。）

### JDBC
```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="${jdbc.driverClassName}" />
	<property name="url" value="${jdbc.url}" />
	<property name="username" value="${jdbc.username}" />
	<property name="password" value="${jdbc.password}" />
</bean>

<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
</bean>
```

### JTA
```xml
<jee:jndi-lookup id="dataSource" jndi-name="jdbc/jpetstore"/>
<bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager" />
```

## Hibernate 事务的配置
也可以轻松使用 Hibernate 本地事务。需要定义一个 Hibernate `LocalSessionFactoryBean` ，应用程序代码可以用它来获取 Hibernate `Session` 实例。
```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
	<property name="dataSource" ref="dataSource"/>
	<property name="mappingResources">
		<list>
			<value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
		</list>
	</property>
	<property name="hibernateProperties">
		<value>
			hibernate.dialect=${hibernate.dialect}
		</value>
	</property>
</bean>

<bean id="txManager" class="org.springframework.orm.hibernate5.HibernateTransactionManager">
	<property name="sessionFactory" ref="sessionFactory"/>
</bean>
```

如果使用 Hibernate 和 Jakarta EE 容器管理的 JTA 事务，则应使用 `JtaTransactionManager` 。此外，建议 Hibernate 通过其事务协调器以及可能的连接释放模式配置来感知 JTA：
```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
	<property name="dataSource" ref="dataSource"/>
	<property name="mappingResources">
		<list>
			<value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
		</list>
	</property>
	<property name="hibernateProperties">
		<value>
			hibernate.dialect=${hibernate.dialect}
		</value>
	</property>
	<property name="jtaTransactionManager" ref="txManager"/>
</bean>

<bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

```
SQLExceptionTranslator
TransactionSynchronizationManage
```