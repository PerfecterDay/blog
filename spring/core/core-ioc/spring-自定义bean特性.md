# 自定义 Bean 特性
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html

Spring框架提供了多种接口，可用于自定义Bean的特性。本节将这些接口按以下方式分组：
+ Lifecycle Callbacks
+ `ApplicationContextAware` and `BeanNameAware`
+ Other `Aware` Interfaces

## Lifecycle Callbacks
要与容器对 Bean 生命周期的管理进行交互，可以实现 Spring 的 `InitializingBean` 和 `DisposableBean` 接口。容器会在初始化时调用前者（ `InitializingBean` ）的 `afterPropertiesSet()` 方法，在销毁时调用后者（ `DisposableBean` ）的 `destroy()` 方法，从而让 Bean 在初始化和销毁时执行特定操作。

在现代Spring应用中，使用 `JSR-250` 的 `@PostConstruct` 和 `@PreDestroy` 注解通常被视为接收生命周期回调的最佳实践。采用这些注解可以使Bean无需与Spring专属接口耦合。

在内部，Spring框架通过 `BeanPostProcessor` 来处理所有可识别的回调接口，并调用相应的方法。若需实现Spring默认未提供的自定义功能或其他生命周期行为，开发者可自行实现 `BeanPostProcessor` 。

除了 bean 本身初始化和销毁回调外，Spring管理的对象还可以实现 `Lifecycle` 接口，使得 bean 能够参与到容器的生命周期的特定阶段处理逻辑。


### Initialization 回调
`org.springframework.beans.factory.InitializingBean` 接口允许容器在为 Bean 设置所有必要属性后，让 Bean 执行初始化工作。 `InitializingBean` 接口定义了一个方法：
```
void afterPropertiesSet() throws Exception;
```
建议不要使用 `InitializingBean` 接口，因为它会使代码与Spring产生耦合。替代方案是:
1. 使用 `@PostConstruct` 注解
2. 或指定POJO初始化方法
   1. 对于基于XML的配置元数据，可通过 `init-method` 属性指定具有 **`void` 无参签名的方法名称**。
   2. 使用 java 配置时，使用 `@Bean` 注解的 `initMethod` 属性

以下几种配置方式能达到相同的效果：
```
public class ExampleBean implements InitializingBean{
    @Override
	public void afterPropertiesSet() {
		// do some initialization work
	}
}

public class ExampleBean{
    @PostConstruct
    void init(){
        ...
    }
}


<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>

@Configuration
public class AppConfig {

	@Bean(initMethod = "init")
	public ExampleBean beanOne() {
		return new ExampleBean();
	}
}
```
当直接使用Java 编程配置时，可以随心所欲地操作对象，而无需始终依赖容器的生命周期管理。

请注意， `@PostConstruct` 注解方法及常规初始化方法均在容器的单例创建锁内执行。只有当 `@PostConstruct` 方法返回后，bean 实例才被视为完全初始化完毕并准备好发布给其他组件。此类独立初始化方法仅用于验证配置状态，并可能根据给定配置准备某些数据结构，但不得进行任何涉及外部 bean 访问的操作。否则将存在初始化死锁风险。

### Destruction 回调
实现 `org.springframework.beans.factory.DisposableBean` 接口可使 bean 在包含它的容器被销毁时获得回调。 `DisposableBean` 接口定义了一个方法：
```
void destroy() throws Exception;
```

类似地，我们推荐使用 `@PreDestroy` 注解，或者在 XML 中使用 `<bean/>` 元素的 `destroy-method` 属性。当使用 java 配置时，使用 `@Bean` 注解的 `destroyMethod` 属性。
```
public class AnotherExampleBean implements DisposableBean {
	@Override
	public void destroy() {
		// do some destruction work (like releasing pooled connections)
	}
}


public class AnotherExampleBean {
	@PreDestroy
	public void preDestroy() {
		// do some destruction work (like releasing pooled connections)
	}
}

<bean id="exampleDestructionBean" class="examples.ExampleBean" destroy-method="cleanup"/>
public class AnotherExampleBean {
	public void cleanup() {
		// do some destruction work (like releasing pooled connections)
	}
}


@Configuration
public class AppConfig {

	@Bean(destroyMethod = "cleanup")
	public ExampleBean beanOne() {
		return new ExampleBean();
	}
}
```

默认情况下，使用Java配置且具有 `public close` 或 `shutdown` 方法的Bean会自动关联一个销毁回调。若定义了 `public close` 或 `shutdown` 方法，但不希望容器关闭时调用该方法，可在Bean定义中添加 `@Bean(destroyMethod = "")` 注解以禁用默认（推断）模式。

### 默认的 Initialization 和 Destroy 方法
当编写初始化与销毁回调方法时，若不使用Spring特有的 `InitializingBean` 和 `DisposableBean` 回调接口，通常会采用 `init()` 、`initialize()` 、 `dispose()` 等命名规范。理想情况下，此类生命周期回调方法的命名应在项目范围内标准化，确保所有开发人员使用统一的方法名称以保持一致性。

然后可以配置 Spring 容器，使其在每个 Bean 上“查找” 规范命名的初始化与销毁回调方法名称。这意味着作为应用程序开发者，只需编写应用程序类并使用名为 `init()` 的初始化回调，而无需在每个 Bean 定义中配置 `init-method="init"` 属性。Spring IoC容器会在创建Bean时调用该方法（并遵循前文所述的标准生命周期回调契约）。此特性同时**强制要求初始化与销毁回调方法采用统一的命名规范。**

```
<beans default-init-method="init" default-destroy-method="dispose">

	<bean id="blogService" class="com.something.DefaultBlogService">
		<property name="blogDao" ref="blogDao" />
	</bean>

</beans>
```
上述配置下，如果 Bean 中定义了 `init()` 方法，就会在初始化后被调用。 bean 销毁时， `dispose` 方法会被调用。

Spring容器保证在为Bean提供所有依赖后，立即调用已配置的初始化回调。因此，初始化回调是在原始Bean引用上调用的，这意味着AOP拦截器等尚未应用于该Bean。目标Bean会先被完整创建，随后才应用带有拦截器链的AOP代理。若目标 bean 与代理分别定义，代码甚至可绕过代理直接操作原始目标 bean。因此若将拦截器应用于  `init` 方法将导致不一致：此举会将目标 bean 的生命周期与其代理或拦截器耦合，当代码直接操作原始目标 bean 时将产生异常语义。

### 回调方法组合使用时的场景
Spring 2.5 之后，我们可以使用以下方式配置 bean 的生命周期：
1. The `InitializingBean` and `DisposableBean` callback interfaces
2. Custom init() and destroy() methods, 就是使用 `initMethod` 或者 `destroyMethod`
3. The `@PostConstruct` and `@PreDestroy` annotations

如果同时使用了上述多种方式来配置同一个 bean 的生命周期回调，那么：
1. 多个方式配置的回调方法不同，那么会按照指定的顺序被调用（后文会介绍）
2. 如果配置的是一个回调方法，那么回调方法只会被调用一次

初始化时，回调方法的调用顺序如下：
1. `@PostConstruct` 注解的方法最先被执行
2. `InitializingBean` 接口的 `afterPropertiesSet()` 被调用
3. 自定义的 `init()` 方法

为什么是这个顺序呢 ？主要看 `AbstractAutowireCapableBeanFactory` 的 `initializeBean` 方法:
```
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    ....
    invokeAwareMethods(beanName, bean); // 首先调用 aware 相关的方法
    ...
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName); // CommonAnnotationBeanPostProcessor 会处理 @PostCOnstruct 注解
    ...
    invokeInitMethods(beanName, wrappedBean, mbd); // 先调用 `InitializingBean.afterPropertiesSet()` 方法，再调用自定义方法 
    ...
    return wrappedBean;
}
```
由上可知： `BeanPostProcessor` 是在 `InitializingBean.afterPropertiesSet()` 方法之前执行。

销毁时的顺序如下：
1. `@PreDestroy` 注解方法先调用
2. `DisposableBean` 接口的 `destroy()` 方法被调用
3. 自定义的 `destroy()` 方法被调用

### Startup and Shutdown Callbacks - LifeCycle
[Spring Lifecycle与SmartLifecycle](/spring/core/core-ioc/spring-lifecycle.md)


### Non-Web Applications 的优雅关闭
Spring基于Web的应用上下文实现已内置代码，可在相关Web应用程序关闭时优雅地关闭Spring IoC容器。这里，我们讨论一下非 Web 应用的优雅关闭。

若在非Web应用环境（例如富客户端桌面环境）中使用Spring的IoC容器，需要向JVM注册一个关闭钩子。此操作可确保容器优雅关闭，并调用单例Bean的相关销毁方法以释放所有资源。但是我们需正确配置并实现这些销毁回调。

要注册关机钩子，请调用 `ConfigurableApplicationContext` 接口上声明的 `registerShutdownHook()` 方法，如下例所示：
```
public static void main(String[] args) {
    ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(Start.class);
    context.registerShutdownHook();
//        context.close();
}
```

如果注释掉 `context.registerShutdownHook();` 且没有显式调用 `context.close();` 方法，那么销毁的回调方法不会被调用。实验效果如下：
注册了 `shutdownHook` 时，会打印所有的销毁回调：
```
postConstruct
afterPropertiesSet
init
preDestroy
DisposableBean destroy
myDestroy
```

注释掉时，只会打印初始化回调：
```
postConstruct
afterPropertiesSet
init
```

以下是一个关于生命周期与 `LifeCycle` 的完整实例：
```
public class A implements Lifecycle, InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet");
    }

    @PostConstruct
    public void postConstruct(){
        System.out.println("postConstruct");
    }

    public void destroy(){
        System.out.println("DisposableBean destroy");
    }

    public void myDestroy(){
        System.out.println("myDestroy");
    }

    @PreDestroy
    public void preDestroy(){
        System.out.println("preDestroy");
    }

    public void init(){
        System.out.println("init");
    }

    @Override
    public void start() {
        System.out.println("lifecycle start");
        isRunning= true;
    }

    @Override
    public void stop() {
        System.out.println("lifecycle stop");
        isRunning= false;
    }

    boolean isRunning = false;
    @Override
    public boolean isRunning() {
        return isRunning;
    }
}


@SpringBootApplication
public class Start {

    @Bean(initMethod = "init",destroyMethod = "myDestroy")
    public A a(){
        return new A();
    }

    public static void main(String[] args) {
        ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(Start.class);
        context.registerShutdownHook();
        context.start();
        context.stop();
    }
}
```

输出：
```
postConstruct
afterPropertiesSet
init
lifecycle start
lifecycle stop
preDestroy
DisposableBean destroy
myDestroy
```

### 线程安全与可见性
> https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html#beans-factory-thread-safety

## Aware 接口
+ `ApplicationContextAware` : 获取 `ApplicationContext` 。
+ `ApplicationEventPublisherAware` : 发布事件
+ `BeanClassLoaderAware` : 用于加载 bean 类的类加载器。
+ `BeanFactoryAware` : 获取 BeanFactory
+ `BeanNameAware` : 获取 BeanName 
+ `LoadTimeWeaverAware` : Defined weaver for processing class definition at load time.
+ `MessageSourceAware` : Configured strategy for resolving messages (with support for parameterization and internationalization).
+ `NotificationPublisherAware` : Spring JMX 通知发布器。
+ `ResourceLoaderAware` : 为低级访问资源配置加载器。
+ `ServletConfigAware` : 容器运行时的当前 `ServletConfig` 。仅在网络感知的 Spring ApplicationContext 中有效。
+ `ServletContextAware` : 容器运行的当前 `ServletContext` 。仅在网络感知的 Spring ApplicationContext 中有效。

使用这些接口会将您的代码与 Spring API 联系在一起，并且不遵循反转控制风格。因此建议只在需要以编程方式访问容器的基础架构 Bean 时使用这些接口。更好的方式是使用自动注入将需要的对象注入到 bean 中。
