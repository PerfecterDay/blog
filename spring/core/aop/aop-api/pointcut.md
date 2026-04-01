# Pointcut API in Spring
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/pointcuts.html

## 概念
Spring 的切点模型支持在不同类型增强中复用切点。可以使用同一个切点来定位不同的增强。

`org.springframework.aop.Pointcut` 接口是核心接口，用于将增强定位到特定的类和方法。该接口的完整定义如下：
```
public interface Pointcut {
	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();
}
```
将 `Pointcut` 接口拆分为两部分，既能复用类和方法匹配部分，又能进行细粒度的组合操作（例如与另一个方法匹配器执行“并集”操作）。

`ClassFilter` 接口用于将切点限制在给定的一组目标类上。如果 `matches()` 方法始终返回 `true` ，则表示匹配所有目标类。以下代码片段展示了 `ClassFilter` 接口的定义：
```public interface ClassFilter {
	boolean matches(Class clazz);
}
```

通常来说， `MethodMatcher` 接口更为重要。该接口的完整定义如下：
```
public interface MethodMatcher {
	boolean matches(Method m, Class<?> targetClass);

	boolean isRuntime();

	boolean matches(Method m, Class<?> targetClass, Object... args);
}
```

`matches(Method, Class)` 方法用于测试该切点是否曾与目标类上的某个给定方法匹配。该评估可在创建 AOP 代理时进行，从而避免在每次方法调用时都需要进行测试。如果针对某个方法调用的双参数的 `matches` 方法返回 `true`，且 `MethodMatcher` 的 `isRuntime()` 方法返回 `true`，则每次方法调用时都会调用三参数的 `matches` 方法。这使得切点能够在目标增强开始执行之前，立即查看传递给该方法调用的参数。

大多数 `MethodMatcher` 的实现都是静态的，这意味着它们的 `isRuntime()` 方法会返回 `false` 。在这种情况下，那个带三个参数的 `matches` 方法将永远不会被调用。

如果可能的话，请尽量将切点设为静态，这样当创建 AOP 代理时，AOP 框架就能缓存切点评估的结果。

## Operations on Pointcuts
Spring 支持对切点进行运算（特别是并集和交集）。

“并集”指任一切点匹配的方法；“交集”指两个切点均匹配的方法。通常情况下，并集更为实用。可以通过调用 `org.springframework.aop.support.Pointcuts` 类中的静态方法，或使用同一包中的 `ComposablePointcut` 类来组合切点。不过，使用 `AspectJ` 切点表达式通常是一种更简单的方法。

## AspectJ Expression Pointcuts
自 2.0 版本以来，Spring 使用的最重要的切点类型是 `org.springframework.aop.aspectj.AspectJExpressionPointcut` 。这是一个利用 AspectJ 提供的库来解析 AspectJ 切点表达式字符串的切点。

## 快捷的切点实现
Spring 提供了多种便捷的切点实现。其中一些可以直接使用；另一些则旨在供用户在特定应用场景中进行子类化。

### 静态切点
静态切点基于方法和目标类，不会考虑方法的参数。对于大多数使用场景而言，静态切点已足够——且是最佳选择。Spring 仅在方法首次被调用时评估静态切点一次。此后，无需在每次方法调用时重新评估该切点。

#### 正则表达式切点
指定静态切点的明显方法之一是使用正则表达式。除了 Spring 之外，还有其他几个 AOP 框架也支持这种方式。 `org.springframework.aop.support.JdkRegexpMethodPointcut` 是一个通用的正则表达式切点，它利用了 JDK 中的正则表达式支持。

使用 `JdkRegexpMethodPointcut` 类时，可以提供一组模式字符串。如果其中任何一个匹配，则切点评估结果为 `true` 。（因此，生成的切点实际上是指定模式的并集。）

```
@Configuration
public class JdkRegexpConfiguration {
	@Bean
	public JdkRegexpMethodPointcut settersAndAbsquatulatePointcut() {
		JdkRegexpMethodPointcut pointcut = new JdkRegexpMethodPointcut();
		pointcut.setPatterns(".*set.*", ".*absquatulate");
		return pointcut;
	}
}
```

Spring 提供了一个名为 `RegexpMethodPointcutAdvisor` 的便利类，它允许我们同时引用一个 `Advice` （请记住， `Advice` 可以是 `interceptor` 、before 增强、throws 增强等）。在后台，Spring 使用的是 `JdkRegexpMethodPointcut` 。使用 `RegexpMethodPointcutAdvisor` 可以简化配置，因为该 Bean 同时封装了切点和增强，如下例所示：
```
@Configuration
public class RegexpConfiguration {

	@Bean
	public RegexpMethodPointcutAdvisor settersAndAbsquatulateAdvisor(Advice beanNameOfAopAllianceInterceptor) {
		RegexpMethodPointcutAdvisor advisor = new RegexpMethodPointcutAdvisor();
		advisor.setAdvice(beanNameOfAopAllianceInterceptor);
		advisor.setPatterns(".*set.*", ".*absquatulate");
		return advisor;
	}
}
```
可以将 `RegexpMethodPointcutAdvisor` 与任何类型的 `Advice` 配合使用。

#### Attribute-driven Pointcuts
元数据驱动的切点是静态切点的一种重要类型。它利用元数据属性的值（通常是源代码级元数据）。

### 动态切点
动态切点的评估成本（匹配计算逻辑）高于静态切点。它们不仅考虑静态信息，还会考虑方法参数。这意味着每次调用方法时都必须对其进行评估，且由于参数会发生变化，因此无法缓存评估结果。就是说动态切点会在每次方法调用时，根据方法的参数值信息实时计算是否匹配。

#### Control Flow Pointcuts
Spring 的控制流切点在概念上与 AspectJ 的 `cflow` 切点相似，尽管功能稍弱。（目前尚无法指定某个切点在另一个切点匹配的连接点之后执行。）控制流切点匹配当前的调用栈。例如，如果连接点是由 `com.mycompany.web` 包中的某个方法或 `SomeCaller` 类调用的，则该切点可能会触发。控制流切点通过 `org.springframework.aop.support.ControlFlowPointcut` 类进行定义。

与其他动态切点相比，控制流切点在运行时进行评估的开销要大得多。在 Java 1.4 中，其开销约为其他动态切点的五倍。


## 切点超类
Spring 提供了有用的切点超类，以帮助用户实现自定义切点。

由于静态切点最为实用，Spring 提供了 `StaticMethodMatcherPointcut` 类。这只需实现一个抽象方法（尽管您可以重写其他方法来定制行为）即可快速定义自己的切点。以下示例演示了如何继承 `StaticMethodMatcherPointcut` 类：
```
class TestStaticPointcut extends StaticMethodMatcherPointcut {
	public boolean matches(Method m, Class targetClass) {
		// return true if custom criteria match
	}
}
```

## 自定义切点
由于 Spring AOP 中的切点是 Java 类，而非语言特性（如 AspectJ 中的情况），因此，用户可以声明自定义切点，无论是静态的还是动态的。Spring 中的自定义切点可以具有任意复杂的结构。不过，如果条件允许， `advice` 使用 AspectJ 切点表达式语言。
