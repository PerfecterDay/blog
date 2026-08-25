# 一文精通 Spring ApplicationContext
{docsify-updated}

## AbstractApplicationContext
```
public void refresh() throws BeansException, IllegalStateException {
        .......
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }
    ....
    }
}
```

### prepareRefresh
```
protected void prepareRefresh() {
    // Switch to active.
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    ......
    // Initialize any placeholder property sources in the context environment.
    initPropertySources();

    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();

    // Store pre-refresh ApplicationListeners...
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    }
    else {
        // Reset local application listeners to pre-refresh state.
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Allow for the collection of early ApplicationEvents,
    // to be published once the multicaster is available...
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```
该方法主要功能如下：
1. 设置相应的状态标志
2. 初始化属性占位符
3. 校验属性配置
4. 配置一些 listeners

主要涉及的一些核心接口/类有 `Environment` , `ConfigurableEnvironment` , `PropertySources` , `PropertySource` 等与环境配置相关的抽象。

### prepareBeanFactory
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
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
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
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
}
```
主要是为 `BeanFactory` 配置一系列需要用到的组件，然后注册了一些 `BeanPostProcessor` 实例，并且通过 `registerSingleton` 手动注册了一些 bean .

+ `registerSingleton(name, instance)`：注册一个真正的 bean（有名字，受 `BeanPostProcessor` 、生命周期、`getBean(name)` 等管理）。
+ `registerResolvableDependency(type, value)`：不是 bean，没有名字，不走生命周期，仅在按类型解析时作为额外候选可见；`getBean(...)`、`getBeansOfType(...)` 看不到它。
+ `ignoreDependencyType` / `ignoreDependencyInterface`：相反方向，把某些 setter 从自动装配视野中移除。

### postProcessBeanFactory
```
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```
这是一个扩展点，继承 `AbstractApplicationContext` 的子类可以重写这个方法，实现更多的 `BeanFactory` 处理。如 `GenericWebApplicationContext` 就重写了该方法：
```
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    if (this.servletContext != null) {
        beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext));
        beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    }
    WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
    WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext);
}
```

### invokeBeanFactoryPostProcessors
```
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
```
主要是调用 `BeanFactoryPostProcessor.postProcessBeanFactory(beanfactory)` 处理 beanfactory . 主要关注 `PostProcessorRegistrationDelegate` 这个类中的调用细节，其中会有 `BeanDefinitionRegistryPostProcessor` 的处理。

### registerBeanPostProcessors
主要完成 `BeanPostProcessor` 类型 bean 的生成与注册：
```
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```
注意， `BeanPostProcessor` 自身也可能会被其他 `BeanPostProcessor` 处理，但只会被"在它之前已经注册"的 `BeanPostProcessor` 处理，而不是被所有的 `BeanPostProcessor` 处理。  
因为 `BeanPostProcessor` 自身也是通过 `beanFactory.getBean()` 创建出来的，所以它会走完整的 Bean 创建生命周期（实例化 → 属性填充 → 初始化）。在初始化阶段，Spring 会调用 `applyBeanPostProcessorsBeforeInitialization` 和 `applyBeanPostProcessorsAfterInitialization`。 

另一点需要注意的是 `PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);` 内部会按照一定顺序注册 `BeanPostProcessor` .

### initMessageSource
注册实例化 `MessageSource` 对象。

### initApplicationEventMulticaster
注册实例化 `ApplicationEventMulticaster` 对象，该对象的主要职责就是发布 Spring Events.

### onRefresh
这也是一个扩展点，子类可以重写这个方法，在 `ApplicationContext` 准备好我们注册的常规 bean 之前做一些操作：
```
/**
* Template method which can be overridden to add context-specific refresh work.
* Called on initialization of special beans, before instantiation of singletons.
*/
protected void onRefresh() throws BeansException {
    // For subclasses: do nothing by default.
}
```

比如， `ServletWebServerApplicationContext` 会在这个方法中启动 `WebServer` 内嵌的服务器：
```
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```

### registerListeners
添加 `ApplicationListener` 类型的 beanName 到 `ApplicationEventMulticaster` 中，注意这里不会生成 bean，只是保存 bean 的名字。

### finishBeanFactoryInitialization
```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Mark current thread for singleton instantiation with applied bootstrap locking.
    beanFactory.prepareSingletonBootstrap();

    // Initialize bootstrap executor for this context.
    if (beanFactory.containsBean(BOOTSTRAP_EXECUTOR_BEAN_NAME) &&
            beanFactory.isTypeMatch(BOOTSTRAP_EXECUTOR_BEAN_NAME, Executor.class)) {
        beanFactory.setBootstrapExecutor(
                beanFactory.getBean(BOOTSTRAP_EXECUTOR_BEAN_NAME, Executor.class));
    }

    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no BeanFactoryPostProcessor
    // (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Call BeanFactoryInitializer beans early to allow for initializing specific other beans early.
    String[] initializerNames = beanFactory.getBeanNamesForType(BeanFactoryInitializer.class, false, false);
    for (String initializerName : initializerNames) {
        beanFactory.getBean(initializerName, BeanFactoryInitializer.class).initialize(beanFactory);
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        try {
            beanFactory.getBean(weaverAwareName, LoadTimeWeaverAware.class);
        }
        ......
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
}
```
这个方法最后调用了 `beanFactory.preInstantiateSingletons();` 方法，这会让 `BeanFactory` 预先创建出所有的 `singleton` 类型的 bean。

### finishRefresh
```
protected void finishRefresh() {
    // Reset common introspection caches in Spring's core infrastructure.
    resetCommonCaches();

    // Clear context-level resource caches (such as ASM metadata from scanning).
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    initLifecycleProcessor();

    // Propagate refresh to lifecycle processor first.
    getLifecycleProcessor().onRefresh();

    // Publish the final event.
    publishEvent(new ContextRefreshedEvent(this));
}
```
1. 清理缓存
2. 初始化 `LifecycleProcessor` : 如果容器中注册了 `lifecycleProcessor` 名字的 `LifecycleProcessor` 类型 bean ，就是用这个 bean ；否则 `registerSingleton(...)` 方法直接注册使用 `DefaultLifecycleProcessor` .
3. 调用 `LifecycleProcessor` 的 `onRefresh` 方法来处理 `Lifecycle` 回调 : `DefaultLifecycleProcessor` 会获取 `Lifecycle` 类型的 bean， 并视情况调用 `Lifecycle.start(...)`
4. 发布 `ContextRefreshedEvent` 事件

关于 `Lifecycle` 请参考 [LifeCycle](/spring/core/core-ioc/spring-lifecycle.md)