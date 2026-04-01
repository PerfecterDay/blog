# @AspectJ 启用与声明
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/aspectj-support.html   
> https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/at-aspectj.html

`@AspectJ` 指将切面声明为带有注解的常规 Java 类的编程风格。该风格由 `AspectJ` 项目在 `AspectJ 5` 版本中引入。Spring 解析的注解与 `AspectJ 5` 完全一致，其点切解析与匹配功能依赖 `AspectJ` 提供的库实现。但其 AOP 运行时仍纯粹基于 Spring AOP，不依赖 AspectJ 编译器或织入器。

## 启用 @AspectJ 
要在 Spring 配置中使用 `@AspectJ` 切面，需要启用 Spring 对基于 `@AspectJ` 切面配置 Spring AOP 的支持，并根据 bean 是否被这些切面增强来自动代理 bean。所谓自动代理，是指当 Spring 确定某个 bean 被一个或多个切面增强时，它会自动为该 bean 生成一个代理，以拦截方法调用并确保增强按需执行。

可通过编程方式或XML配置启用 `@AspectJ` 支持。无论采用何种方式，都需要确保 `AspectJ` 的 `org.aspectj:aspectjweaver` 库（版本1.9或更高）位于应用程序的类路径中。
```
@Configuration
@EnableAspectJAutoProxy
public class ApplicationConfiguration {
}
```

```
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
			https://www.springframework.org/schema/beans/spring-beans.xsd
			http://www.springframework.org/schema/aop
			https://www.springframework.org/schema/aop/spring-aop.xsd">

	<aop:aspectj-autoproxy />
</beans>
```

## 声明一个 Aspect
启用 `@AspectJ` 支持后，Spring会自动检测应用上下文中所有由 `@AspectJ` 切面类（带有 `@Aspect` 注解）定义的Bean，并用于配置Spring AOP。以下两个示例:
```
@Aspect
public class NotVeryUsefulAspect {
}
```

切面（用 `@Aspect` 注解标记的类）可以包含方法和字段，与其他类相同。它们还可包含 `pointcut` , `advice` , 和 `introduction` 声明。

可以在 Spring XML 配置中将切面类注册为常规 Bean，或者通过 `@Configuration` 类中的 `@Bean` 方法进行注册，或让 Spring 通过类路径扫描自动检测它们——这与任何其他 Spring 管理的 Bean 相同。但需注意：仅添加 `@Aspect` 注解不足以实现类路径自动检测。为此，您需要额外添加 `@Component` 注解（或根据Spring组件扫描器的规则，使用符合要求的自定义符号注解）。

在Spring AOP中，切面本身不能成为其他切面增强的目标。类上的 `@Aspect` 注解将其标记为切面，会将其排除在自动代理之外。