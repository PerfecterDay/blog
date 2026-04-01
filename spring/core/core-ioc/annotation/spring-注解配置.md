# 基于注解的配置
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config.html

Spring 提供了基于注解的配置的全面支持，通过在相关类、方法或字段声明上使用注解，可以直接提供 bean 定义、自动注入、配置等原数据信息。Spring 结合使用 `BeanPostProcessors` 与注解，使核心 IOC 容器能够识别特定注解。

`@Autowired` 注解提了[spring-autowiring](/spring/core//core-ioc/dependencies/spring-autowiring.md) 中描述的类似功能，但具备更精细的控制能力和更广泛的应用场景。此外，Spring还支持 `JSR-250` 注解（如 `@PostConstruct` 和 `@PreDestroy` ），以及包含在 `jakarta.inject` 包中的 `JSR-330`（Java依赖注入）注解（如 `@Inject` 和 `@Named` ）。

注解注入发生在 XML/Groovy 等外部配置注入之前。因此，当通过混合方式进行配置时，外部配置（例如XML指定的Bean属性）会有效覆盖属性的注解。

 `AnnotationConfigApplicationContext` 会在启动时隐式注册一些 `BeanPostProcessors` 以支持注解功能。

使用 XML 时：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

</beans>
```

The `<context:annotation-config/>` 元素会注册以下 post-processors:
+ `ConfigurationClassPostProcessor`
+ `AutowiredAnnotationBeanPostProcessor`
+ `CommonAnnotationBeanPostProcessor`
+ `PersistenceAnnotationBeanPostProcessor`
+ `EventListenerMethodProcessor`


`@Autowired` 、 `@Inject` 、 `@Value` 和 `@Resource` 注解由 Spring `BeanPostProcessor` 实现处理。这意味着您不能在自定义的 `BeanPostProcessor` 或 `BeanFactoryPostProcessor` 类型（如有）中应用这些注解。

这些类型必须通过 XML 或 Spring 的 `@Bean` 方法显式进行配置。