# AOP 代理机制
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/core/aop/proxying.html

Spring AOP 通过 JDK 动态代理或 CGLIB 为指定目标对象创建代理。JDK 动态代理内置于 JDK 中，而 `CGLIB` 则是常见的开源类定义库（已重新打包至 `spring-core` 中）。

若待代理的目标对象至少实现了一个接口，则使用JDK动态代理，此时目标类型实现的所有接口均会被代理。若目标对象未实现任何接口，则创建 `CGLIB` 代理——该代理是目标类型的运行时生成的子类。

若需强制使用 `CGLIB` 代理（例如对目标对象定义的所有方法进行代理，而不仅限于其接口实现的方法），可采取此操作。但需注意以下事项：
+ `final` 类无法被代理，因为它们不可继承。
+ `final` 方法无法被增强，因为它们不可覆盖。
+ `private` 方法无法被增强，因为它们不可覆盖。
+ 不可见的方法（例如不同包中父类的包私有方法）无法被增强，因为它们实质上是私有方法。
+ 由于 `CGLIB` 代理实例是通过 `Objenesis` 创建的，被代理对象的构造函数不会被调用两次。但若JVM不支持构造函数跳过机制，可能会看到重复调用及Spring AOP支持生成的相应调试日志条目。
+ 在Java模块系统环境中使用 `CGLIB` 代理可能受限。典型场景是部署在模块路径时，无法为 `java.lang` 包的类创建 `CGLIB` 代理。此类情况需使用JVM引导标志参数 `--add-opens=java.base/java.lang=ALL-UNNAMED` ，但该标志不适用于模块环境。

## 强制使用特定AOP代理类型
```
<aop:config proxy-target-class="true">
	<!-- other beans defined here... -->
</aop:config>

<aop:aspectj-autoproxy proxy-target-class="true"/>
```

`@EnableAspectJAutoProxy` 、 `@EnableTransactionManagement` 及相关配置注解均提供对应的 `proxyTargetClass` 属性。这些注解同样被整合为单一的统一自动代理创建器，在运行时有效应用最强的代理设置。自 7.0 版本起，此机制同样适用于独立代理处理器（例如 `@EnableAsync` ），确保在特定应用程序中所有自动代理尝试均遵循统一的全局默认设置。

全局默认代理类型可能因配置而异。虽然核心框架默认增强使用基于接口的代理，但Spring Boot可能根据配置属性默认启用基于类的代理。

从7.0版本起，可通过在特定 `@Bean` 方法或 `@Component` 类上添加 `@Proxyable` 注解，强制为单个Bean指定特定代理类型。此时使用 `@Proxyable(INTERFACES)` 或 `@Proxyable(TARGET_CLASS)` 将覆盖全局配置的默认值。针对特殊需求，甚至可通过 `@Proxyable(interfaces=…)` 指定要使用的代理接口，从而将暴露范围限定为目标Bean实现的特定接口，而非全部接口。

## 理解 AOP 代理
**Spring AOP 基于代理实现**。在编写自定义切面或使用 Spring 框架提供的任何基于 Spring AOP 的切面之前，理解这句话的实际含义至关重要。

```
public class SimplePojo implements Pojo {

	public void foo() {
		// this next method invocation is a direct call on the 'this' reference
		this.bar();
	}

	public void bar() {
		// some logic...
	}
}
```
若在对象引用上调用方法，该方法将直接在该对象引用上调用，如下图和代码所示：
<center><img src="pics/aop-proxy-plain-pojo-call.png" alt=""></center>

```
public class Main {

	public static void main(String[] args) {
		Pojo pojo = new SimplePojo();
		// this is a direct method call on the 'pojo' reference
		pojo.foo();
	}
}
```


当客户端代码引用的对象是代理时，情况会略有不同。请参考下图及代码片段：
<center><img src="pics/aop-proxy-call.png" alt=""></center>

```
public class Main {

	public static void main(String[] args) {
		ProxyFactory factory = new ProxyFactory(new SimplePojo());
		factory.addInterface(Pojo.class);
		factory.addAdvice(new RetryAdvice());

		Pojo pojo = (Pojo) factory.getProxy();
		// this is a method call on the proxy!
		pojo.foo();
	}
}
```


这里的关键在于， `Main` 类中 `main(..)` 方法内的客户端代码持有代理对象的引用。这意味着对该对象引用的方法调用实际上是对代理的调用。因此，代理能够将调用委托给所有与该方法调用相关的 `interceptor` （增强）。然而，一旦调用最终到达目标对象（本例中为 `SimplePojo` 引用），该对象对自身的任何方法调用（如 `this.bar()` 或 `this.foo()` ）都将针对 `this` 引用而非代理进行执行。这具有重要意义：**它意味着自我调用不会触发方法调用关联的增强运行**。换言之，通过显式或隐式 `this` 引用进行的自我调用将绕过增强。

为解决此问题，可选择以下方案：
1. 禁止自我调用
最佳方案（此处"最佳"一词使用较为宽松）是重构代码以避免自我调用。这确实需要付出一些努力，但这是最优且侵入性最小的解决方案。

2. 注入自我引用
另一种方法是利用 [`self`注入](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html#beans-autowired-annotation-self-injection)，通过 `self` 引用而非 `this` 来调用代理上的方法。

3. 使用 `AopContext.currentProxy()`
最后一种方法强烈不 `advice` 采用，我们甚至犹豫是否要提及它，更推荐前面的方案。但作为最后手段，你可以选择将类内部的逻辑与Spring AOP绑定，如下例所示：
```
public class SimplePojo implements Pojo {
	public void foo() {
		// This works, but it should be avoided if possible.
		((Pojo) AopContext.currentProxy()).bar();
	}

	public void bar() {
		// some logic...
	}
}
```

使用 `AopContext.currentProxy()` 会使代码完全耦合到 Spring AOP，并让类本身意识到它正处于 AOP 上下文中，这会削弱 AOP 部分优势。此外，它还要求 `ProxyFactory` 配置为暴露代理，如下例所示：
```
public class Main {
	public static void main(String[] args) {
		ProxyFactory factory = new ProxyFactory(new SimplePojo());
		factory.addInterface(Pojo.class);
		factory.addAdvice(new RetryAdvice());
		factory.setExposeProxy(true);

		Pojo pojo = (Pojo) factory.getProxy();
		// this is a method call on the proxy!
		pojo.foo();
	}
}
```



