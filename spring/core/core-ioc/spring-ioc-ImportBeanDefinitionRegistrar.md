# BeanFactoryPostProcessor 与 ImportBeanDefinitionRegistrar 自定义控制 bean 定义
{docsify-updated}

## BeanFactoryPostProcessor
```
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

`BeanFactoryPostProcessor` 的实现会在 ApplicationContext 的 `refresh()` 方法的特定阶段被调用。

## ImportBeanDefinitionRegistrar
```
public interface ImportBeanDefinitionRegistrar {
	/**
	 * Register bean definitions as necessary based on the given annotation metadata of
	 * the importing {@code @Configuration} class.
	 */
	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
			BeanNameGenerator importBeanNameGenerator) {

		registerBeanDefinitions(importingClassMetadata, registry);
	}

	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	}
}
```
`ImportBeanDefinitionRegistrar` 需要与 `@import` 配合使用，要在一个配置类或者 Bean 中 import 相应的 `ImportBeanDefinitionRegistrar` 实现。参数 `importingClassMetadata` 就是使用 `@Import` 导入 `ImportBeanDefinitionRegistrar` 的配置类或 bean 。  
由源码可以看出， `ImportBeanDefinitionRegistrar` 本质上是一个接口。在 `ImportBeanDefinitionRegistrar` 接口中，有一个 `registerBeanDefinitions()` 方法，通过 `registerBeanDefinitions()` 方法，我们可以向Spring容器中注册bean实例。

Spring官方在动态注册bean时(大多数时候是为一些特殊注解生成代理 bean)，大部分套路其实是使用 `ImportBeanDefinitionRegistrar` 接口。

所有实现了该接口的类都会被 `ConfigurationClassPostProcessor` 处理， `ConfigurationClassPostProcessor` 实现了 `BeanFactoryPostProcessor` 接口，所以 `ImportBeanDefinitionRegistrar` 中动态注册的bean是优先于依赖其的bean初始化的，也能被aop、validator等机制处理。