# @Aspect 注解风格的AOP配置
{docsify-updated}

`@AspectJ` 指的是一种将切面声明为带有注解的常规 Java 类的样式的配置方式。作为 AspectJ 5 版本的一部分，AspectJ 项目引入了 `@AspectJ` 风格。Spring 使用 AspectJ 提供的库对注解进行解析和匹配，从而支持与 AspectJ 5 相同的注解。不过，AOP 运行时仍然是纯粹的 Spring AOP，不依赖于 AspectJ 编译器或编织器。

## 启用 @Aspect 注解支持
要在 Spring 配置中使用 `@AspectJ` 切面需要启用 Spring 支持，以便根据 `@AspectJ` 切面配置 Spring AOP，并根据 Bean 是否被这些切面Advice自动代理。自动代理是指，如果 Spring 确定某个 Bean 受一个或多个切面的 Advice ，它会自动为该 Bean 生成一个代理来拦截方法调用，并确保根据需要运行 Advice 。

@AspectJ 支持可通过 XML 或 Java 式配置启用。无论哪种情况，都需要确保 AspectJ 的 `aspectjweaver.jar` 库位于应用程序的类路径上（1.9 或更高版本）。该库可从 AspectJ 发行版的 lib 目录或 Maven Central 资源库中获取。

要使用 Java `@Configuration` 启用 @AspectJ 支持，添加 `@EnableAspectJAutoProxy` 注解，如下例所示：
```
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
```

## 定义切面
启用 @AspectJ 支持后，Spring 会自动检测应用程序上下文中定义的任何 Bean，并将具有 `@Aspect` 注解配置的类使用 Spring AOP。
```
@Aspect
public class MyAspect {
}
```
与其他类一样，Aspects（带 @Aspect 注解的类）可以有方法和字段。它们还可以包含切点、Advice和 introduction 声明。

需要注意的是 `@Aspect` 注解并不会被自动扫描扫描，因此如果切面要被自动扫描还需要加上 `@Component` 之类的注解 。在 Spring AOP 中，切面本身不能成为其他切面 Advice 的目标。类上的 `@Aspect` 注解将其标记为一个切面，并因此将其排除在自动代理之外。

## 定义切点
切点确定了兴趣连接点，从而使我们能够控制Advice的运行时间。Spring AOP 仅支持 Spring Bean 的方法执行连接点，切点济就是选择匹配 Spring Bean 上的特定方法。  
一个切点声明包含两个部分：
+ 一个由名称和任何参数组成的签名
+ 一个切点表达式，切点表达式确定我们感兴趣的方法执行。

在 AOP 的 `@AspectJ` 注解风格中，方法签名和常规方法定义完全一样，但是返回类型必须是 void 。而切点表达式则通过使用 `@Pointcut` 注解来表示。
```
@Pointcut("execution(* transfer(..))") // the pointcut expression
private void anyOldTransfer() {} // the pointcut signature
```

### 切点标志符(designator)
AspectJ5的切点表达式由标志符(designator)和操作参数组成. 如 `execution( greetTo(..))` 的切点表达式, `execution` 就是标志符, 而圆括号里的 `greetTo(..)` 就是操作参数.

Spring AOP 支持在切点分表达式中使用以下 AspectJ 切点标志符 (PCD)：
1. `execution` : 匹配 join point 的执行, 例如 `execution(* hello(..))` 表示匹配所有目标类中的 hello() 方法. 这个是最基本的 pointcut 标志符.
2. `within`: 匹配特定包下的所有 join point, 例如:
   + `within(com.xys.*)` 表示 `com.xys` 包中的所有连接点, 即包中的所有类的所有方法. 
   + `within(com.xys.service.*Service)` 表示在 `com.xys.service` 包中所有以 Service 结尾的类的所有方法.
3. `this` : 的作用是匹配一个 bean, 这个 bean(Spring AOP proxy) 是一个给定类型的实例(instance of). 
4. `target`: 匹配的是一个目标对象(target object, 即需要织入 advice 的原始的类), 此原始对象是一个给定类型的实例(instance of).
5. `bean` : 匹配 bean 名字为指定值的 bean 下的所有方法,这是spring 特有的，并不包含在AspectJ之内。
	+ `bean(*Service)` // 匹配名字后缀为 Service 的 bean 下的所有方法
	+ `bean(myService)` // 匹配名字为 myService 的 bean 下的所有方法
6. `args` : 匹配参数满足要求的的方法，另外还可以将方法参数传递到 Advice 方法参数。
7. `@annotation`: 匹配由指定注解所标注的方法, 例如:
	```
	@Pointcut("@annotation(com.xys.demo1.AuthChecker)")
	public void pointcut() {
	}
	```
    则匹配由注解 AuthChecker 所标注的方法.
8. `@target/@args/@within` : 与上述对应的标志符类似，只是是注解的形式。

完整的 AspectJ 切点语言支持 Spring 不支持的其他切点标志符：`call、get、set、preinitialization、staticinitialization、initialization、handler、adviceexecution、withincode、cflow、cflowbelow、if、@this` 和 `@withincode`。在 Spring AOP 解释的切点分表达式中使用这些切点分代号会导致抛出 `IllegalArgumentException` 异常。

### 标志符参数
```java
execution(modifiers-pattern?
			ret-type-pattern
			declaring-type-pattern?name-pattern(param-pattern)
			throws-pattern?)
```
除了返回类型 ret-type-pattern 、name-pattern 和 param-pattern ，其他部分都是可选的。

+ `modifiers-pattern` - 方法的可见性（public、protected、private、*） ret-type-pattern - 方法的返回类型
+ `ret-type-pattern` - 方法的返回值类型
+ `declaring-type-pattern` - 包或类，`com.app.service.*` - 适用于该包中的所有类，`com.app.service.UserService` - 仅适用于 UserService 类，\* - 全部）
+ `name-pattern` - 方法名称（例如：`set*` - 匹配所有 set 开头的方法）
+ `param-pattern` - 限定方法参数，(\*) 模式匹配只接受一个任意类型参数的方法。(\*,String)匹配接收两个参数的方法。第一个参数可以是任何类型，而第二个参数必须是字符串
+ `throws-pattern` - 引发此异常的方法。


## 定义 Advice 增强
Advice 与切点表达式相关联，可以有 before/after/around 类型。切点表达式既可以是内联切点，也可以是对已命名切点的引用。

### Before Advice
```java
@Aspect
public class BeforeExample {

	@Before("execution(* com.xyz.dao.*.*(..))")
	public void doAccessCheck() {
		// ...
	}

	@Before("com.xyz.CommonPointcuts.dataAccessOperation()") //引用@Pointcut定义的切点
	public void doAccessCheck() {
		// ...
	}
}
```

### After Returning Advice
有时，您需要在 Advice 中访问方法返回的实际值。可以使用 `@AfterReturning` 绑定返回值的形式来获取该访问权限，如下例所示：
```java
@Aspect
public class AfterReturningExample {

	@AfterReturning(
		pointcut="execution(* com.xyz.dao.*.*(..))",
		returning="retVal")
	public void doAccessCheck(Object retVal) {
		// ...
	}
}
```
`returning` 属性指定了返回值应该绑定到 Advice 方法的参数名。

### AfterThrowing Advice
```java
@Aspect
public class AfterThrowingExample {

	@AfterThrowing(
		pointcut="execution(* com.xyz.dao.*.*(..))",
		throwing="ex")
	public void doRecoveryActions(DataAccessException ex) {
		// ...
	}
}
```
`throwing` 属性指定了抛出的异常绑定到 Advice 方法的参数名。

### After (Finally) Advice
```java
@Aspect
public class AfterFinallyExample {

	@After("execution(* com.xyz.dao.*.*(..))")
	public void doReleaseLock() {
		// ...
	}
}
```

### Around Advice
最后一种Advice是 Around Advice。Around Advice "环绕 "匹配方法的执行而运行。它有机会在方法运行之前和之后执行工作，并决定方法何时、如何甚至是否真正运行。如果需要以线程安全的方式共享方法执行前后的状态，例如启动和停止计时器，通常会使用 Around Advice。

通过使用 `@Around` 注解对方法进行注解，可声明 Around Advice。方法应声明为 `Object` 为其返回类型，方法的第一个参数必须是 `ProceedingJoinPoint` 类型。在Advice方法的主体中，必须在 `ProceedingJoinPoint` 上调用 `proceed()` 才能运行真正的底层方法。在不带参数的情况下调用 `proceed()` 将导致在调用底层方法时提供调用者的原始参数。对于高级用例，`proceed()` 方法有一个重载变体，可接受参数数组 `Object[]` 。在调用底层方法时，数组中的值将被用作该方法的参数。

`proceed()` 可以被调用一次/多次或者不调用，当不调用时，真正的方法就不会被调用。
```java
@Aspect
public class AroundExample {

	@Around("execution(* com.xyz..service.*.*(..))")
	public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
		// start stopwatch
		Object retVal = pjp.proceed();
		// stop stopwatch
		return retVal;
	}
}
```

### 在 Advice 中获取真正方法的参数
1. 使用 `JoinPoint`

	任何 advice 方法都可以声明一个 `org.aspectj.lang.JoinPoint` 类型的参数作为它的第一个参数。需要注意的是，Around Advice 需要声明 `ProceedingJoinPoint` 类型的第一个参数，而 `ProceedingJoinPoint` 是 `JoinPoint` 的子类。

	JoinPoint 接口提供了许多有用的方法：
	+ `getArgs()` ：获取方法参数。
	+ `getThis()`：获取代理对象。
	+ `getTarget()`：获取目标对象。
	+ `getSignature()`：获取真正执行方法的描述。
	+ `toString()`：打印被 `advice` 方法的有用描述。

2. 使用 `args`
   ```
   @Before("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
	public void validateAccount(Account account) {
		// ...
	}

	@Pointcut("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
	private void accountDataAccessOperation(Account account) {}

	@Before("accountDataAccessOperation(account)")
	public void validateAccount(Account account) {
		// ...
	}
   ```
   pointcut 表达式中的 `args(account,...)` 部分有两个作用。
   + 首先，它将匹配范围限制在方法至少需要一个参数，且传递给该参数的参数是 Account 实例的情况下。
   + 其次，它通过 account 参数向 `advice` 提供实际的 Account 对象。

## 定义 Introductions
Introductions 可以给为一个指定对象动态指定实现一个接口，并且为其提供一个默认的接口实现。 可以使用 `@DeclareParents` 注解进行介绍。该注解用于声明匹配类型有一个新的父类（因此得名）。
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