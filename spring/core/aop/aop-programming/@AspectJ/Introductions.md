# Introductions
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/introductions.html

`Introductions` （在AspectJ中称为类型间声明）使切面能够声明被切入对象实现了某个接口，并代表这些对象提供该接口的实现。

可以使用 `@DeclareParents` 注解声明 `Introduction` 。该注解用于声明匹配的类型具有新的父类型（故得此名）。例如，假设存在一个名为 `UsageTracked ` 的接口及其名为 `DefaultUsageTracked` 的实现，以下切面声明所有服务接口的实现者同时实现 `UsageTracked` 接口（例如用于通过 JMX 进行统计）：
```
@Aspect
public class UsageTracking {
	@DeclareParents(value="com.xyz.service.*+", defaultImpl=DefaultUsageTracked.class)
	public static UsageTracked mixin;

	@Before("execution(* com.xyz..service.*.*(..)) && this(usageTracked)")
	public void recordUsage(UsageTracked usageTracked) {
		usageTracked.incrementUseCount();
	}
}
```
要实现的接口由注解字段的类型决定，比如上面 `@DeclareParents` 注解修饰的是 `UsageTracked` 接口类型的字段，则要实现的就是 `UsageTracked` 接口。`@DeclareParents` 注解的 `value` 属性是一个 `AspectJ` 类型模式。任何匹配类型的Bean都实现了 `UsageTracked` 接口。请注意，在上例的 `before` 增强中， `service` 包下的Bean可直接作为 `UsageTracked` 接口的实现。若需通过编程方式访问Bean，则应编写如下代码：
```
UsageTracked usageTracked = context.getBean("myService", UsageTracked.class);
```