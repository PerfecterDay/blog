# Spring cloud 动态刷新 bean 魔法- RefreshScope
{docsify-updated}

## RefreshScope的使用
当配置发生变更时，标记为 `@RefreshScope` 的Spring `@Bean` 会获得特殊处理。此特性解决了有状态 `Bean` 仅在初始化时才注入配置的问题。例如，当通过环境变量修改数据库URL时，若数据源存在已打开的连接，通常需要允许这些连接持有者完成当前操作。但是，下次从连接池借用连接时，即可获得使用新URL的连接。

有时，对于某些只能初始化一次的Bean，甚至必须应用 `@RefreshScope` 注解。如果Bean是“不可变”的，则必须为其添加 `@RefreshScope` 注解，或在 `spring.cloud.refresh.extra-refreshable` 属性键下指定类名。

**Refresh scope beans是惰性的，当你在某个服务中注入了 `@RefreshScope` 类型的 bean 时，该 bean 并不会立即被注入，注入的实际上是一个代理对象，当你在调用 bean 的某个方法时，代理对象会去尝试获取 bean，如果bean 没有初始化，那么 bean 会被初始化并且`RefreshScope` 会缓存初始化好的 bean 。当调用了 `RefreshScope` 刷新方法后，会将缓存中的 bean 删除，这样当再次调用 bean 方法时，代理对象就会重新初始化它们，这样就能使用最新的初始化值。**

`RefreshScope` 是上下文中的一个 Bean，其公开的 `refreshAll()` 方法通过清除目标缓存来刷新作用域内的所有 `Bean` 。 `/refresh` 端点通过 HTTP 或 JMX 协议暴露此功能。若需按名称刷新单个 Bean，还可使用 `refresh(String)` 方法。

要暴露 `/refresh` 端点，您需要在应用程序中添加以下配置：
```
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

refresh 端点： `curl -XPOST http://localhost:8091/actuator/refresh`


测试实例：
```
@Component
@RefreshScope
@Data
public class B {

    @Value("${cms.url}")
    private String cmsUrl;
}

@Resource
    B b;

@GetMapping("/aaa")
public String aa(){
    Map<String, Object> map = new HashMap<>();
    map.put("cms.url", "new-value"); // 修改或新增属性

    // 添加到最前面，优先级最高
    MapPropertySource propertySource = new MapPropertySource("dynamic-properties", map);
    environment.getPropertySources().addFirst(propertySource);

    HashSet set = new HashSet();
    set.add("cms.url");
    applicationContext.publishEvent(new EnvironmentChangeEvent(set));
    return b.getCmsUrl();
}
```

当你执行 `curl http://localhost:8050/user/aaa` 时，即使刷新了配置值，依旧返回的是初始化时的值。只有执行了 `curl -XPOST http://localhost:8091/actuator/refresh` 后，再次执行 `curl http://localhost:8050/user/aaa` 才会返回新值。


### 原理
**1. 代理类的生成**

`ClassPathBeanDefinitionScanner` 会将注解 `@RefreshScope` 的 bean 的 beanDefinition 重写： 
```
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            // 收集 Scope 信息
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition abstractBeanDefinition) {
                postProcessBeanDefinition(abstractBeanDefinition, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition annotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations(annotatedBeanDefinition);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                // 处理 @RefreshScope 注解的 beanDefinition
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}

关键调用链：
definitionHolder =AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
-> ScopedProxyCreator.createScopedProxy(...) -> ScopedProxyUtils.createScopedProxy(...)
```

在处理 `BeanDefinition` 的时候就会生成一个 `beanClass` 属性改为 `ScopedProxyFactoryBean` 的代理类型的 bean。 而将原始的 baan 改名为 `scopedTarget.xxx` 。
<center><img src="pics/refreshScope1.png" alt=""></center>

更进一步， `GenericScope` 类会进一步将 beanClass 改为 `GenericScope.LockedScopedProxyFactoryBean` 的类型。至此完成偷天换日。
```
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    for (String name : registry.getBeanDefinitionNames()) {
        BeanDefinition definition = registry.getBeanDefinition(name);
        if (definition instanceof RootBeanDefinition root) {
            if (root.getDecoratedDefinition() != null && root.hasBeanClass()
                    && root.getBeanClass() == ScopedProxyFactoryBean.class) {
                if (getName().equals(root.getDecoratedDefinition().getBeanDefinition().getScope())) {
                    root.setBeanClass(LockedScopedProxyFactoryBean.class);
                    root.getConstructorArgumentValues().addGenericArgumentValue(this);
                    // surprising that a scoped proxy bean definition is not already
                    // marked as synthetic?
                    root.setSynthetic(true);
                }
            }
        }
    }
}
```

综上所述，当你使用 `@RefreshScope` 注解声明一个 bean 时， Spring 会为其生成代理类 `GenericScope.LockedScopedProxyFactoryBean` 的 bean:
```
GenericScope.LockedScopedProxyFactoryBean -> ScopedProxyFactoryBean

@Override
public void setBeanFactory(BeanFactory beanFactory) {
    if (!(beanFactory instanceof ConfigurableBeanFactory cbf)) {
        throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
    }
    this.scopedTargetSource.setBeanFactory(beanFactory);

    ProxyFactory pf = new ProxyFactory();
    pf.copyFrom(this);
    pf.setTargetSource(this.scopedTargetSource);

    Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
    Class<?> beanType = beanFactory.getType(this.targetBeanName);
    if (beanType == null) {
        throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
                "': Target type could not be determined at the time of proxy creation.");
    }
    if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
        pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
    }

    // Add an introduction that implements only the methods on ScopedObject.
    ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
    pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

    // Add the AopInfrastructureBean marker to indicate that the scoped proxy
    // itself is not subject to auto-proxying! Only its target bean is.
    pf.addInterface(AopInfrastructureBean.class);

    this.proxy = pf.getProxy(cbf.getBeanClassLoader());
}

@Override
public Object getObject() {
    if (this.proxy == null) {
        throw new FactoryBeanNotInitializedException();
    }
    return this.proxy;
}
```

**2. 动态代理bean增强执行的过程**

debug 看调用栈帧：
<center><img src="pics/refresh.png" alt=""></center>

`SimpleBeanTargetSource` 的 `getTarget()` 方法如下：
```
@Override
public Object getTarget() throws Exception {
    return getBeanFactory().getBean(getTargetBeanName());
}
```

`BeanFactory` 在生产 bean 时，会根据 beandefinition 的信息来获取对应的 scope，不同的 scope 使用不同的 `Scope` 实现类来生成 bean 对象，一些常见的 `Scope` 实现类如下：
<center><img src="pics/scope.png" alt=""></center>

```
AbstractBeanFactory -> doGetBean :
Object scopedInstance = scope.get(beanName, () -> {
    beforePrototypeCreation(beanName);
    try {
        return createBean(beanName, mbd, args);
    }
    finally {
        afterPrototypeCreation(beanName);
    }
});
```

其中 `RefreshScope` 继承自 `GenericScope` ，其 `get` 方法如下：
```
public Object get(String name, ObjectFactory<?> objectFactory) {
    BeanLifecycleWrapper value = this.cache.put(name, new BeanLifecycleWrapper(name, objectFactory));
    this.locks.putIfAbsent(name, new ReentrantReadWriteLock());
    try {
        return value.getBean();
    }
    catch (RuntimeException e) {
        this.errors.put(name, e);
        throw e;
    }
}
```
Cache 结构如下：
<center><img src="pics/refresh_scope.png" alt=""></center>

真正实现 `@RefreshScope` bean 创建的是 `BeanLifecycleWrapper` ，其中会保存 bean 实例对象，如果实例对象已经创建就直接返回该对象，如果没有创建就会创建一个新的实例对象。

### 总结
Spring 会为 `@RefreshScope` 标注的bean 生成一个代理对象，调用代理对象的方法时，每次会去会去 `Beanfactory` 中获取 bean 实例， `Beanfactory` 对于不同的 scope 使用不同的 `Scope` 实现类来生成 bean 实例，其中 `RefreshScope` 继承自 `GenericScope` ，`GenericScope` 的 `get` 方法会从 `GenericScope` 中获取 bean 实例，如果存在就返回该实例，如果实例不存在，就会创建一个新的实例对象并且缓存下来，下次再调用该方法时，就直接从缓存中获取。
当配置刷新时（`RefreshScope.refreshAll()`）, `GenericScope` 中缓存的 bean 就会被清空，下次调用时就会重新创建一个新的实例对象。这时候 `Beanfactory` 会重新执行一遍 bean 实例化的过程，从而刷新最新配置。
