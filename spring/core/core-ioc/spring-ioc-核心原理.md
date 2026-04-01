#  Spring IOC 核心原理
 {docsify-updated}
> https://docs.spring.io/spring-framework/reference/core/beans.html

<center><img src="pics/spring-bean-lifecycle.webp" alt=""></center>

Spring的IoC容器所起的作用就是它会以某种方式加载Configuration Metadata（通常也就是XML格式的配置信息），然后根据这些信息绑定整个系统的对象，最终组装成一个可用的基于轻量级容器的应用系统。
Spring的IoC容器实现以上功能的过程，基本上可以按照类似的流程划分为两个阶段，即
1. **容器启动阶段**-对应图5-1中的第一步
2. **Bean实例化阶段**。-对应图5-1中的第二步
<center>
<img src="pics/spring-ioc0.png" height=40% width=40%>
<img src="pics/ioc-phase.png" height=40% width=40%>
</center>

Spring的IoC容器在实现的时候，充分运用了这两个实现阶段的不同特点，在每个阶段都加入了相应的容器扩展点，以便我们可以根据具体场景的需要加入自定义的扩展逻辑。

## 配置元数据加载阶段
容器启动伊始，首先会通过某种途径加载 Configuration MetaData 。除了代码方式比较直接，在大部分情况下，容器需要依赖某些工具类（ `BeanDefinitionReader` ）对加载的 Configuration MetaData 进行解析和分析，并将分析后的信息编组为相应的 `BeanDefinition` ，最后把这些保存了bean定义必要信息的 `BeanDefinition` ，注册到相应的 `BeanDefinitionRegistry` ，这样容器启动工作就完成了。  
总地来说，该阶段所做的工作可以认为是准备性的，重点更加侧重于对象管理信息的收集。当然，一些验证性或者辅助性的工作也可以在这个阶段完成。

### 第一阶段的扩展点 BeanFactoryPostProcessor
Spring提供了一种叫做 `BeanFactoryPostProcessor` 的容器扩展机制。该机制允许我们在容器实例化相应对象之前，对注册到容器的 `BeanDefinition` 所保存的信息做相应的修改。这就相当于在容器实现的第一阶段最后加入一道工序，让我们对最终的 `BeanDefinition` 做一些额外的操作，比如修改其中bean定义的某些属性，为bean定义增加其他信息等。

`org.springframework.context.support.PropertySourcesPlaceholderConfigurer` 和 `org.springframework.beans.factory.config.PropertyOverrideConfigurer` 是两个比较常用的 `BeanFactoryPostProcessor` 。另外，为了处理配置文件中的数据类型与真正的业务对象所定义的数据类型转换， Spring还允许我们通过 `org.springframework.beans.factory.config.CustomEditorConfigurer` 来注册自定义的 `PropertyEditor` 以补充容器中默认的 `PropertyEditor` 。

## Bean实例化阶段  
当调用`getBean()`时，容器会首先检查所请求的对象之前是否已经存在。如果没有，则会根据注册的 `BeanDefinition` 所提供的信息实例化被请求对象，并为其注入依赖。  
如果该对象实现了某些回调接口，也会根据回调接口的要求来装配它。当该对象装配完毕之后，容器会立即将其返回请求方使用。  
如果说第一阶段只是根据图纸装配生产线的话，那么第二阶段就是使用装配好的生产线来生产具体的产品了。

### 第二阶段扩展 BeanPostProcessor
在已经可以借助于 `BeanFactoryPostProcessor` 来干预 Spring 的第一个阶段启动之后，我们就可以开始探索下一个阶段，即bean实例化阶段的实现逻辑了。
```
package scripting;
import org.springframework.beans.factory.config.BeanPostProcessor;
public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

	// 在bean实例化之前，这里可以对 bean 进行处理
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean; // we could potentially return any object reference here...
	}

	// bean 实例化之后
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		System.out.println("Bean '" + beanName + "' created : " + bean.toString());
		return bean;
	}
}
```

Spring 的 `AutowiredAnnotationBeanPostProcessor` 就是实现 `BeanPostProcessor` 来实现 `@Autowired` 注解功能的。

### Bean的生命周期
容器启动之后，并不会马上就实例化相应的bean定义。我们知道，容器现在仅仅拥有所有对象的 BeanDefinition 来保存实例化阶段将要用的必要信息。只有当请求方通过 BeanFactory 的 `getBean()` 方法来请求某个对象实例的时候，才有可能触发Bean实例化阶段的活动。 

BeanFactory 的 `getBean()` 方法可以被客户端对象显式调用，也可以在容器内部隐式地被调用。隐式调用有如下两种情况:
+ 对于BeanFactory来说，对象实例化默认采用延迟初始化。通常情况下，当对象A被请求而需要第一次实例化的时候，如果它所依赖的对象B之前同样没有被实例化，那么容器会先实例化对象A所依赖的对象。这时容器内部就会首先实例化对象B，以及对象 A依赖的其他还没有实例化的对象。这种情况是容器内部调用getBean()，对于本次请求的请求方是隐式的。
+ ApplicationContext启动之后会实例化所有的bean定义，这个特性在本书中已经多次提到。但ApplicationContext在实现的过程中依然遵循Spring容器实现流程的两个阶段，只不过它会在启动阶段的活动完成之后，紧接着调用注册到该容器的所有bean定义的实例化方法getBean()。这就是为什么当你得到ApplicationContext类型的容器引用时，容器内所有对象已经被全部实例化完成。不信你查一下类org.AbstractApplicationContext的refresh()方法。

#### BeanFactory中bean的生命周期

<center><img src="pics/beanfactory-bean-lifecycle.png" height=40% width=40%></center>

BeanFactory 实例化一个 `bean` 的步骤：
<center><img src="pics/spring-bean.jpg" width=80%></center>

注意到：自定义的 `bean` 实例化时，会调用 `BeanPostProcessor`相关的方法，`BeanPostProcessor` 不一定是 `bean` 本身实现的接口，它是 Spring 容器提供的容器级别接口，所有实现 `BeanPostProcessor` 的类在注册到容器中后，都会在 `spring` 实例化某个`bean`的时候其作用。就是说，如果我们自定义了一个 `BeanPostProcessor` 的实现类并注册到容器中，则它会在`spring`实例化所有其它`bean`的时候起作用。

#### ApplicationContext中bean的生命周期
<center><img src="pics/applicationContext-bean-lifecycle.png" height=40% width=40%></center>

Bean 在应用上下文中的生命周期与在 BeanFactory 中的类似，实际上，应用上下文在初始化时，会向容器中注册一个 `ApplicationContextAwareProcessor` 类型的 `BeanPostProcessor` 实现类，并实现 `postProcessBeforeInitialization` 方法，这样的话，调用 `getBean`方法时， `ApplicationContextAwareProcessor` 就会生效， `postProcessBeforeInitialization` 判断如果 Bean 实现了 `EnvironmentAware` , `EmbeddedValueResolverAware` , `ResourceLoaderAware` , `ApplicationEventPublisherAware` , `MessageSourceAware` , `ApplicationContextAware` 接口，则会分别调用它们。

ApplicationContext 和 BeanFactory 的一个重大区别在于：前者会利用java反射机制自动识别出注册的 `BeanPostProcessor`, `BeanFactoryPostProcessor` , `InstantiationAwareBeanPostProcessor` ,并自动将它们注册到容器中；而后者需要手工调用 `addBeanPostProcessor` 方法注册它们。

<center><img src="pics/bean-postprocessors.png" alt=""></center>

`ApplicationContext` 的 `refresh` 方法，会在 `ApplicationContext` 初始化时调用：
```
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备从context
			prepareRefresh();

			// 获取beanFactory，此时会解析加载xml中的beanDefinition，但是并没有注册 bean 对象
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 准备 beanFactory
			prepareBeanFactory(beanFactory);

            // 调用子类（实现ApplicationContext接口的子类）postProcessBeanFactory 复写的方法，允许实现类处理 beanfactory
            postProcessBeanFactory(beanFactory);

            // 注册并调用BeanFactoryPostProcessor的postProcessBeanFactory方法，第一扩展点
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册 BeanPostProcessor 类型的bean，自动实现 BeanPostProcessor 注册，第二扩展点注册
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // 实例化所有 bean，在这里边会去调用 BeanPostProcessor 第二扩展点接口对生成的 bean 实例进行处理
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
    }
```

#### bean 实例化的策略模式 InstantiationStrategy
```
AbstractAutowireCapableBeanFactory 的方法中使用策略框架bean：
protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
	try {
		Object beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
	}
}
```
Spring 提供了两种策略实现: `SimpleInstantiationStrategy` 和 `CglibSubclassingInstantiationStrategy`。`CglibSubclassingInstantiationStrategy`继承自 `SimpleInstantiationStrategy`。
查看 `SimpleInstantiationStrategy` 的源码：
```
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
		final Constructor<?> ctor, Object... args) {

	if (!bd.hasMethodOverrides()) {
		if (System.getSecurityManager() != null) {
			// use own privileged to change accessibility (when security is on)
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				ReflectionUtils.makeAccessible(ctor);
				return null;
			});
		}
		return BeanUtils.instantiateClass(ctor, args);
	}
	else {
		return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
	}
}
```
可以发现，如果 RootBeanDefinition 中有方法需要重写就会使用 `instantiateWithMethodInjection` 方法实例化，否则就直接调用 `BeanUtils.instantiateClass` 方法实例化。  
`CglibSubclassingInstantiationStrategy` 通过重写 `instantiateWithMethodInjection` 方法实现自定义 Cglib 实例化。

从 `AbstractAutowireCapableBeanFactory` 的构造函数中可以看出，默认情况下，只要不是 graalvm 的本地映像， BeanFactory 使用的策略是 `CglibSubclassingInstantiationStrategy`。
```
public AbstractAutowireCapableBeanFactory() {
	super();
	ignoreDependencyInterface(BeanNameAware.class);
	ignoreDependencyInterface(BeanFactoryAware.class);
	ignoreDependencyInterface(BeanClassLoaderAware.class);
	if (NativeDetector.inNativeImage()) {
		this.instantiationStrategy = new SimpleInstantiationStrategy();
	}
	else {
		this.instantiationStrategy = new CglibSubclassingInstantiationStrategy();
	}
}
```

#### bean创建的一些主要方法过程
 
看看几个主要的方法：

1. `AbstractApplicationContext` 的 `obtainFreshBeanFactory()` 方法 :

	```
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}

	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		//加载beanDefinition
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	```
	该方法主要是实例化一个 `beanFactory`，并加载 bean 定义。

2. `AbstractApplicationContext` 的 `prepareBeanFactory`:
```
    protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```
3. 事件相关的监听器注册： `AbstractApplicationContext` 的 `registerListeners`:
```
    protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```