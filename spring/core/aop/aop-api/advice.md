# Advice 和 Advisor API
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/advice.html   
> https://docs.spring.io/spring-framework/reference/core/aop-api/advisor.html

## Advice 的生命周期
每个增强都是一个 Spring Bean。一个增强实例可以在所有被增强对象之间共享，也可以是每个被增强对象对应一个增强对象实例。这分别对应于 `per-class` 和 `per-instance` 类型的增强。

最常使用的是 `per-class` 增强。这种方式适用于通用增强，例如事务代理。此类增强不依赖于被代理对象的状态，也不会添加新的状态，它们仅对方法及其参数进行操作。

按实例的增强适用于引入，以支持混合类。在这种情况下，该增强会向被代理对象添加状态。

## Spring 中的增强类型
Spring 提供了多种增强类型，并且支持扩展以支持任意类型的增强。本节将介绍基本概念和标准增强类型。

### Interception Around Advice
Interception Around Advice 是 Spring 中最基础的增强类型。

Spring 符合 AOP Alliance 针对使用方法拦截的 `around advice` 所定义的接口。因此，实现 `around advice` 的类应实现 `org.aopalliance.intercept` 包中的以下 `MethodInterceptor` 接口：
```
public interface Advice {}
public interface Interceptor extends Advice {}

public interface MethodInterceptor extends Interceptor {
	Object invoke(MethodInvocation invocation) throws Throwable;
}
```

`invoke()` 方法的 `MethodInvocation` 参数可以访问被调用的方法、目标连接点、AOP 代理以及该方法的参数。 `invoke()` 方法应返回调用的结果：通常是连接点的返回值。

```
public class DebugInterceptor implements MethodInterceptor {

	public Object invoke(MethodInvocation invocation) throws Throwable {
		System.out.println("Before: invocation=[" + invocation + "]");
		Object result = invocation.proceed();
		System.out.println("Invocation returned");
		return result;
	}
}
```

请注意对 `MethodInvocation` 的 `proceed()` 方法的调用。这将沿 `interceptor` 链向下执行，直至到达连接点。大多数 `interceptor` 都会调用此方法并返回其返回值。然而， `MethodInterceptor` （如同任何 around 增强一样）可以选择返回不同的值或抛出异常，而不是调用 `proceed` 方法。不过，除非有充分理由，否则不应这样做。

`MethodInterceptor` 的实现提供了与其他符合 AOP Alliance 标准的 AOP 实现之间的互操作性。本节剩余部分讨论的其他增强类型虽然实现了常见的 AOP 概念，但采用的是 Spring 特有的实现方式。虽然使用最具体的增强类型有其优势，但如果你可能需要在其他 AOP 框架中运行该切面， `advice` 在增强类型选择上坚持使用 `MethodInterceptor` 。请注意，目前不同框架之间的切点尚不具备互操作性，且 AOP Alliance 目前尚未定义切点接口。

### Before Advice
`before advice` 的主要优势在于无需调用 `proceed()` 方法，因此不会出现因疏忽而未能沿 `interceptor` 链继续执行的情况。

```
public interface BeforeAdvice extends Advice {}

public interface MethodBeforeAdvice extends BeforeAdvice {
	void before(Method m, Object[] args, Object target) throws Throwable;
}
```

请注意，返回类型为 `void` 。 `before advice` 可以在连接点执行之前插入自定义行为，但无法更改返回值。如果 `before advice` 抛出异常，则会停止 `interceptor` 链的后续执行。该异常会沿 `interceptor` 链向上传播。如果该异常是未检查异常，或者属于被调用方法的签名范围，则会直接传递给客户端。否则，AOP 代理会将其包装为未检查异常。

以下例子展示了 `before advice` 的用法：
```
public class CountingBeforeAdvice implements MethodBeforeAdvice {
	private int count;

	public void before(Method m, Object[] args, Object target) throws Throwable {
		++count;
	}

	public int getCount() {
		return count;
	}
}
```

### Throws Advice
如果连接点抛出了异常，则在连接点返回后调用 `Throws Advice` 。Spring 提供了类型化的`Throws Advice`。请注意，这意味着 `org.springframework.aop.ThrowsAdvice` 接口不包含任何方法。它是一个标记接口，用于标识给定对象实现了一个或多个类型化的`Throws Advice`方法。这些方法应采用以下形式：
```
public interface ThrowsAdvice extends AfterAdvice {}

afterThrowing([Method, args, target], subclassOfThrowable)
```
仅需最后一个参数。方法签名可能包含一个或四个参数，这取决于增强方法是否关注该方法及其参数。接下来的两个代码片段展示了 `Throws Advice` 的示例类。
```
public class RemoteThrowsAdvice implements ThrowsAdvice {
	public void afterThrowing(RemoteException ex) throws Throwable {
		// Do something with remote exception
	}
}

public class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

	public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
		// Do something with all arguments
	}
}

public static class CombinedThrowsAdvice implements ThrowsAdvice {

	public void afterThrowing(RemoteException ex) throws Throwable {
		// Do something with remote exception
	}

	public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
		// Do something with all arguments
	}
}
```

如果一个 `Throws Advice` 方法自身抛出异常，它将覆盖原始异常（即更改抛给用户的异常）。被覆盖的异常通常是 `RuntimeException` ，它与任何方法签名都兼容。然而，如果一个 `Throws Advice` 方法抛出受检查异常，它必须与目标方法声明的异常相匹配，因此，它在某种程度上与特定的目标方法签名相关联。切勿抛出与目标方法签名不兼容的未声明受检查异常！

### After Returning Advice
Spring 中的 `After Returning Advice` 必须实现 `org.springframework.aop.AfterReturningAdvice` 接口，如下所示：
```
public interface AfterReturningAdvice extends Advice {
	void afterReturning(Object returnValue, Method m, Object[] args, Object target)
			throws Throwable;
}
```

返回后的回调函数可以访问返回值（但无法修改它）、被调用的方法、该方法的参数以及目标对象。

示例：
```
public class CountingAfterReturningAdvice implements AfterReturningAdvice {

	private int count;

	public void afterReturning(Object returnValue, Method m, Object[] args, Object target)
			throws Throwable {
		++count;
	}

	public int getCount() {
		return count;
	}
}
```
此增强不会改变执行路径。如果抛出异常，该异常将沿 `interceptor` 链向上抛出，而非作为返回值返回。

### Introduction Advice
`Introduction` 需要一个 `IntroductionAdvisor` 和一个 `IntroductionInterceptor` ，它们需实现以下接口：
```
public interface IntroductionInterceptor extends MethodInterceptor {
	boolean implementsInterface(Class intf);
}
```
从 AOP Alliance `MethodInterceptor` 接口继承的 `invoke()` 方法必须实现引入功能。也就是说，如果被调用方法位于已引入的接口上，则引入 `interceptor` 负责处理该方法调用——它不能调用 `proceed()` 。

`Introduction Advice` 不能与任何切点一起使用，因为它仅在类级别生效，而非方法级别。只能将引入 `advice` 与 `IntroductionAdvisor` 配合使用，该类具有以下方法：
```
public interface IntroductionAdvisor extends Advisor, IntroductionInfo {
	ClassFilter getClassFilter();

	void validateInterfaces() throws IllegalArgumentException;
}

public interface IntroductionInfo {

	Class<?>[] getInterfaces();
}
```

`Introduction Advice` 不包含 `MethodMatcher` ，因此也没有相关的 `Pointcut` 。此时仅支持类过滤。  
`getInterfaces()` 方法返回该 `advice` 者引入的接口。   
`validateInterfaces()` 方法在内部用于检查配置的 `IntroductionInterceptor` 是否能够实现这些引入的接口。

### 自定义 Advice 类型
Spring AOP 设计上具有可扩展性。虽然目前内部采用的是拦截实现策略，但除了 `interception around advice`, `before`, `throws advice`, and `after returning advice` 之外，它还能够支持任意类型的 `Advice` 。

`org.springframework.aop.framework.adapter` 包是一个 SPI 包，它允许在不修改核心框架的情况下添加对新自定义 `Advice` 类型的支持。自定义 `Advice` 类型的唯一限制是，它必须实现 `org.aopalliance.aop.Advice` 标记接口。

## The Advisor API in Spring
Spring 中的 `Advisor` 代表了一个切面，该切面中只包含一个 `Advice` 对象以及其关联的切点表达式。
```
public interface Advisor {
	Advice EMPTY_ADVICE = new Advice() {};
	Advice getAdvice();
	default boolean isPerInstance() {
		return true;
	}
}
```

除了 `introductions` 之外，任意的 `advisor` 可以和任意的 `advice` 一起使用。 

`org.springframework.aop.support.DefaultPointcutAdvisor` 是最常用的 `advisor` . 它可以和 `MethodInterceptor` , `Before Advice` 或者 `Throws Advice` 配合使用。

在 Spring 中，可以在同一个 AOP 代理中混合使用不同类型的 `Advisor` 和 `advice` 。例如，可以在一个代理配置中同时使用 `Around advice` 、 `Throws Advice` 和 `Before Advice` 。Spring 会自动创建所需的 `interceptor` 链。
