# 监听事务事件
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html

从 Spring 4.2 开始，事件监听器可以绑定到事务的某个阶段。典型的应用场景是在事务成功完成时处理该事件。这样做使得当当前事务的结果对监听器确实重要时，事件能够被更灵活地使用。

`@EventListener` 注解来注册常规事件监听器。如果需要监听事务相关的事件，可以使用 `@TransactionalEventListener` 注解。这样做时，监听器默认会绑定到事务的提交阶段，也就是说事务提交后会触发事件监听器的执行。
```java
@Component
public class MyComponent {

	@TransactionalEventListener
	public void handleOrderCreatedEvent(CreationEvent<Order> creationEvent) {
		// ...
	}
}
```

`@TransactionalEventListener` 注解有一个 `phase` 属性，让你可以自定义监听器应绑定的事务阶段。有效的阶段包括 `BEFORE_COMMIT` 、 `AFTER_COMMIT`（默认）、`AFTER_ROLLBACK` 以及 `AFTER_COMPLETION` ，揽括了事务提交的所有情况（无论是提交还是回滚）。