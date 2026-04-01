# Spring 容器扩展点
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html

## Customizing Beans by Using a BeanPostProcessor
`BeanPostProcessor` 接口定义了回调方法，通过实现这些方法可以自定义（或覆盖容器默认的）bean 实例化、依赖解析等逻辑。 `BeanPostProcessor` 实例作用于 bean（或对象）实例。也就是说，Spring IoC 容器先实例化 bean 实例，然后 `BeanPostProcessor` 实例才开始执行其工作。 `BeanPostProcessor` 实例的作用域限定于单个容器。此特性仅在使用容器层次结构时生效。若在某个容器中定义 `BeanPostProcessor` ，它仅对该容器内的 Bean 进行后处理。换言之，即使两个容器同属一个层次结构，定义于其中一个容器的 `BeanPostProcessor` 也不会对另一个容器中定义的 Bean 进行后处理。

如果需要定义多个 `BeanPostProcessor` 实例，并且要按照一定顺序执行，可以同时实现 `Ordered` 的接口，实现 `int getOrder()`  方法指定顺序：**返回值越大，优先级越低。**

`BeanPostProcessor` 接口定义如下：
```
public interface BeanPostProcessor {

    default @Nullable Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

    default @Nullable Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```
`org.springframework.beans.factory.config.BeanPostProcessor` 接口包含两个回调方法。当此类类被注册为容器的后处理器时，对于容器创建的每个 Bean 实例，后处理器都会在容器初始化方法（如 `InitializingBean.afterPropertiesSet()` 或任何声明的 `init` 方法）调用之前，以及任何 Bean 初始化回调之后，收到来自容器的回调。 执行顺序如下：
```
构造方法
    ↓
BeanPostProcessor.postProcessBeforeInitialization
    ↓
InitializingBean.afterPropertiesSet
    ↓
自定义 init-method
    ↓
BeanPostProcessor.postProcessAfterInitialization
```

后处理器可以对 Bean 实例执行任何操作，比如就是用默认实现，原样返回 baan 。 通常 `BeanPostProcessor` 会处理一些回调，或者基于原始 baan 生成一些代理 。Spring AOP基础架构类正是通过实现 `BeanPostProcessor` 来提供代理封装逻辑。

`ApplicationContext` 会自动检测配置元数据中定义的所有实现 `BeanPostProcessor` 接口的 Bean 并将这些 Bean 注册为后处理器，以便在后续创建 Bean 时调用它们。 `BeanPostProcessor` 可以以和其他 Bean 相同的方式部署在容器中。

请注意，当通过配置类上的 `@Bean` 工厂方法声明 `BeanPostProcessor` 时，该工厂方法的返回类型应为实现类本身，或至少为 `org.springframework.beans.factory.config.BeanPostProcessor` 接口，以明确表明该Bean 是一个 `BeanPostProcessor` 类型，不能声明为 `Object` 或者实现的其它接口类型 。否则， `ApplicationContext` 在完全创建该Bean之前无法通过类型自动检测到它。由于 `BeanPostProcessor` 需要在 `ApplicationContext` 中早期实例化才能应用于其他Bean的初始化，这种早期类型检测至关重要。举例如下：
```
public class A extends B implements BeanPostProcessor,OtherInterface {}

@Bean
public A a(){...}  // OK

@Bean
public BeanPostProcessor a(){...}  // OK

@Bean
public B a(){...}  // NOT OK

@Bean
public OtherInterface a(){...}  // NOT OK
```

虽然推荐通过 `ApplicationContext` 自动检测（如前所述）来注册 `BeanPostProcessor` ，但也可以通过 `ConfigurableBeanFactory` 的 `addBeanPostProcessor` 方法进行手动注册。当需要根据条件来判断是否注册，或在层次结构中跨上下文复制 `BeanPostProcessor` 时，此方法尤为实用。但需注意：**程序化添加的 `BeanPostProcessor` 实例不遵循 `Ordered` 接口。此时执行顺序完全取决于注册顺序。而且，程序化注册的 `BeanPostProcessor` 实例始终优先于自动检测注册的实例进行处理，无论是否存在显式排序。**

实现 `BeanPostProcessor` 接口的类具有特殊性，容器对其处理方式不同。所有 `BeanPostProcessor` 实例及其直接引用的 Bean 都会在启动时实例化，这是 `ApplicationContext` 特殊启动阶段的一部分。随后，所有 `BeanPostProcessor` 实例将按排序方式注册，并应用于容器中的后续所有 Bean。由于 AOP 代理本身即以 `BeanPostProcessor` 形式实现，因此 `BeanPostProcessor` 实例及其直接引用的 Bean 均不符合自动代理条件，故不会 AOP 代理增强。

对于任何此类 Bean，通常会看到一条信息日志消息：`Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying)`

若通过自动装配或 `@Resource` 注解将Bean注入到 `BeanPostProcessor` 中时，Spring在搜索类型匹配的依赖项候选时可能访问到意外的Bean，从而导致这些Bean无法进行自动代理或者这些bean 无法被 `BeanPostProcessor` 处理。

关于 `ApplicationContext` 是如何检测以及注册 `BeanPostProcessor` 的详细过程，可以阅读 `AbstractApplicationContext` 的 `registerBeanPostProcessors` 方法：
```
/**
    * Instantiate and register all BeanPostProcessor beans,
    * respecting explicit order if given.
    * <p>Must be called before any instantiation of application beans.
    */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

### AutowiredAnnotationBeanPostProcessor
Spring 的 `AutowiredAnnotationBeanPostProcessor` 是随 Spring 发行版提供的 `BeanPostProcessor` 实现，用于实现 Spring 的自动注入功能。

## BeanFactoryPostProcessor
`org.springframework.beans.factory.config.BeanFactoryPostProcessor`接口的语义与 `BeanPostProcessor` 类似，两者的区别在于： `BeanFactoryPostProcessor` 作用于bean配置元数据。具体而言，Spring IoC容器允许 `BeanFactoryPostProcessor` 读取配置元数据，并在容器实例化除 `BeanFactoryPostProcessor` 实例外的任何bean之前对其进行潜在修改。

```
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
若需修改实际的 Bean 实例，则应改用 `BeanPostProcessor` 。虽然技术上可以在 `BeanFactoryPostProcessor` 中操作bean实例（例如通过调用 `BeanFactory.getBean()` ），但此操作会导致bean过早实例化，违反标准容器的生命周期，绕过bean后处理流程，。这可能引发不可预见的状态。

当 `BeanFactoryPostProcessor` 在 `ApplicationContext` 中声明时，系统会自动运行它，以便将更改应用于定义容器的配置元数据。Spring 包含多个预定义的 Bean Factory 后处理器，例如 `PropertyOverrideConfigurer` 和 `PropertySourcesPlaceholderConfigurer` 。


### ApplicationContext 如何调用 BeanFactoryPostProcessor
请阅读 `AbstractApplicationContext` 的 `invokeBeanFactoryPostProcessors` 方法：
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (for example, through an @Bean method registered by ConfigurationClassPostProcessor)
    if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null &&
            beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}

### PropertySourcesPlaceholderConfigurer
`PropertySourcesPlaceholderConfigurer` 可以将配置文件中的属性注入到 `BeanDefinition` 中。
```
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
	<property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="${jdbc.driverClassName}"/>
	<property name="url" value="${jdbc.url}"/>
	<property name="username" value="${jdbc.username}"/>
	<property name="password" value="${jdbc.password}"/>
</bean>

// properties 文件
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```
运行时， `PropertySourcesPlaceholderConfigurer` 会应用于元数据，替换数据源的某些属性。替换值采用 `${属性名}` 形式的占位符指定，遵循 Ant、log4j 和 JSP EL 的风格。  
`PropertySourcesPlaceholderConfigurer `会检查 `BeanDefinition` 中大多数属性及属性的占位符。我们还可以自定义占位符的前缀、后缀、默认值分隔符和转义字符。此外，可通过JVM系统属性（或 SpringProperties 机制）设置 `spring.placeholder.escapeCharacter.default` 属性，实现全局修改或禁用默认转义字符。

`PropertySourcesPlaceholderConfigurer` 不仅会在指定的 `Properties` 文件中查找属性。默认情况下，如果在指定的属性文件中找不到某个属性，它还会检查 Spring `Environment` 中的属性及 Java `System` 属性

还可以使用 `PropertySourcesPlaceholderConfigurer` 来动态加载类名，这在需要在运行时选择特定实现类时有时会很有用。以下示例展示了如何实现：
```
<bean class="org.springframework.beans.factory.config.PropertySourcesPlaceholderConfigurer">
	<property name="locations">
		<value>classpath:com/something/strategy.properties</value>
	</property>
	<property name="properties">
		<value>custom.strategy.class=com.something.DefaultStrategy</value>
	</property>
</bean>

<bean id="serviceStrategy" class="${custom.strategy.class}"/>
```


### PropertyOverrideConfigurer
`PropertyOverrideConfigurer` 是另一种 `BeanFactoryPostProcessor` ，与 `PropertySourcesPlaceholderConfigurer` 相似，但不同之处在于原始定义中的bean属性可以具有默认值或完全不存在值。若覆盖的 `Properties` 文件中不包含某个bean属性的条目，则使用默认上下文定义。

```
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
```
以上配置会给 `dataSource` bean 的 `driverClassName` 和 `url` 属性分别设置为相应值。


## Customizing Instantiation Logic with a FactoryBean
如果我们需要定一个工厂 bean ，这个 bean 用来生成其它的 bean，则可以实现 `org.springframework.beans.factory.FactoryBean` 接口。  
`FactoryBean` 接口是Spring IoC容器实例化逻辑中的可插拔点。如果需要写复杂的初始化代码，则更适合用Java而非冗长的XML表达时，就可创建自定义 `FactoryBean` ，在该类中编写复杂初始化逻辑，随后将自定义 `FactoryBean` 接入容器。

```
public interface FactoryBean<T> {
    @Nullable T getObject() throws Exception;

    @Nullable Class<?> getObjectType();

    default boolean isSingleton() {
		return true;
	}
}
```

+ ` T getObject()`: 返回需要创建的 bean 对象
+ `Class<?> getObjectType()`: 返回创建的 bean 对象的类型，如果对象类型不可知，则返回 `null`
+ `boolean isSingleton()`: 是否是 `singleton` 类型的 bean。

`FactoryBean` 概念及接口在 Spring 框架的多个位置被使用。Spring 自身就包含了超过 50 个实现 `FactoryBean` 接口的实现类。

当需要向容器请求实际的 `FactoryBean` 实例本身（而非其生成的bean）时，在调用 `ApplicationContext` 的 `getBean()` 方法时，需在 bean 的 id 前添加 `&` 符号。因此，对于ID为 `myBean` 的 `FactoryBean` ，在容器上调用 `getBean("myBean")` 将返回 `FactoryBean` 生成的 bean 实例，而调用 `getBean("&myBean")` 则会返回 `FactoryBean` 实例本身。

