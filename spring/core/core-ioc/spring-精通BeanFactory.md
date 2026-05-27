# 一文精通 Spring BeanFactory
{docsify-updated}

## TL;DR
Spring 的 `BeanFactory` 中获取 bean 时，有几个扩展点：
```
doGetBean : 首先尝试从三级缓存中获取，获取到直接返回；没有则视情况去父 BeanFactory 尝试获取，如果没有，就根据 bean 的 Scope 类型去创建 bean
    createBean : 调用 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(beanClass, beanName) 方法获取 bean，如果有一个 InstantiationAwareBeanPostProcessor 返回非 null 的实例，就返回这个实例， 后续的 InstantiationAwareBeanPostProcessor 不会再调用。 然后再调用 BeanPostProcessor.postProcessAfterInitialization(...) 处理这个实例后返回。如果没有一个 InstantiationAwareBeanPostProcessor 返回实例，则调用 doCreateBean(...) 创建 bean
        doCreateBean : 
            createBeanInstance(beanName, mbd, args) : 1. 如果 RootBeanDefinition 定义了 instanceSupplier ，就调用该 Supplier 生成实例并返回;
                                                      2. 如果 RootBeanDefinition 定义了 FactoryMethodName 工厂方法，就是用工厂方法生成实例并返回
                                                      3. 调用适当的构造函数，配合 InstantiationStrategy 生成实例
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName): 调用 MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(...) 方法
            populateBean(beanName, mbd, instanceWrapper): 1.在依赖注入之前调用 InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(...) 方法处理bean；
                                                          2. 执行依赖注入
                                                          3. 调用 InstantiationAwareBeanPostProcessor.postProcessProperties(....) 处理属性值
                                                          4. 属性绑定
            exposedObject = initializeBean(beanName, exposedObject, mbd)：
                invokeAwareMethods(beanName, bean)：执行 Aware 相关的接口回调
                applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName)：调用 BeanPostProcessor.postProcessBeforeInitialization(...)
                invokeInitMethods(beanName, wrappedBean, mbd): 调用 InitializingBean.afterPropertiesSet()/init-method 方法
                applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)：调用 BeanPostProcessor.postProcessAfterInitialization(...) 方法
```

另外，还有一点需要强调的是，如果直接使用 `BeanFactory` ，要想使用 `BeanPostProcessor` ,  `InstantiationAwareBeanPostProcessor` 这些扩展点，必须手动实例化这些组件实例并且注册到 `BeanFactory` 中才能生效。 `BeanFactoryPostProcessor` 则必须实例化后手动调用处理 `BeanFactory` 才能生效。

## BeanFactory
提供了获取 bean 的接口：
```
public interface BeanFactory {...}

BeanFactory (顶层接口)
 └─ ListableBeanFactory
     └─ ConfigurableListableBeanFactory  ← getBeanFactory() 返回类型
         └─ AbstractBeanFactory
             └─ AbstractAutowireCapableBeanFactory
                 └─ DefaultListableBeanFactory  ← 真正的实现类（最终类）
```

<center><img src="pics/beanfactory.png" alt=""></center>

### doGetBean-AbstractBeanFactory
```
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
    ...
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //保存创建好的 singleton bean 
    private final Set<String> singletonsCurrentlyInCreation = ConcurrentHashMap.newKeySet(16); //记录正在创建的 bean
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16); //Cache of early singleton objects: bean name to bean instance.
	private final Map<String, ObjectFactory<?>> singletonFactories = new ConcurrentHashMap<>(16);
	private final Set<String> alreadyCreated = ConcurrentHashMap.newKeySet(256);

    private final List<BeanPostProcessor> beanPostProcessors = new BeanPostProcessorCacheAwareList();
    private final Map<String, Scope> scopes = new LinkedHashMap<>(8);
    private ApplicationStartup applicationStartup = ApplicationStartup.DEFAULT;
    private final Map<String, RootBeanDefinition> mergedBeanDefinitions = new ConcurrentHashMap<>(256); //最终存放 BeanDefinition 元信息
	private final Map<String, DisposableBean> disposableBeans = new LinkedHashMap<>();
    ...
}
```

核心的 `doGetBean` 方法：
```
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		String beanName = transformedBeanName(name);
		Object beanInstance;
----------
		// Eagerly check singleton cache for manually registered singletons.
        // 尝试从缓存中获取 bean，如果缓存中有，就返回这个 bean
		Object sharedInstance = getSingleton(beanName); //缓存逻辑
		if (sharedInstance != null && args == null) {
			...
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null); //看看是否需要调用工厂方法生成 bean
		}
-----------
// 如果在父 BeanFactory 中有这个 bean，就递归调用父 BeanFactory 的 getBean 方法并返回
		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory abf) {
					return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
----------
			// 前置处理
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName); // this.alreadyCreated.add(beanName); 标记为创建的 bean
			}

			StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
					.tag("beanName", name);
			try {
				if (requiredType != null) {
					beanCreation.tag("beanType", requiredType::toString);
				}
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

---------------
                // 创建依赖的 bean
				// Guarantee initialization of beans that the current bean depends on.

				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							....
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
					}
				}
---------------
				// 创建 Singleton 类型的 bean
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						.....
					});
					beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
---------------
            // 创建 Prototype 类型的 bean
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
---------------
            //创建其他 Scope 类型的 bean
				else {
					String scopeName = mbd.getScope();
					....
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					...
				}
			}
			.....
			finally {
				beanCreation.end();
				if (!isCacheBeanMetadata()) {
					clearMergedBeanDefinition(beanName);
				}
			}
		}

		return adaptBeanInstance(name, beanInstance, requiredType);
	}
```
1. 尝试从缓存的 bean 中获取，手动注册bena可以通过如 `SingletonBeanRegistry.registerSingleton(String beanName, Object singletonObject)` 等方法注册到 `BeanFactory` 中的 bean ，如果缓存中有就直接返回
2. 如果当前 `BeanFactory` 没有 bean ，尝试从父 `BeanFactory` 中获取，如果获取到则返回
3. 如果有依赖的 bean 就先依次创建好 `depends-on` 的 bean
4. 如果是 `singleton` 类型的 bean 则进入创建 `singleton` 流程
5. 如果是 `prototype` 类型的 bean 则进入创建 `prototype` 流程
6. 如果是其他 `Scope` 类型的 bean 则使用指定的 `Scope` 创建bean

4,5,6 步骤最终都会调用到 `createBean(...)` 方法。

### createBean-AbstractAutowireCapableBeanFactory
以下就是 `AbstractAutowireCapableBeanFactory.createBean(...)` 的核心代码:

```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object @Nullable [] args)
        throws BeanCreationException {
	...
    RootBeanDefinition mbdToUse = mbd;
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    ....
---------------
	// 尝试调用 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(...)获取 bean 实例，如果有自定义
	// InstantiationAwareBeanPostProcessor 能返回bean 实例，则调用 BeanPostProcessor.postProcessAfterInitialization(...)
	// 处理生成的 bean，然后直接返回这个 bean
    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    ...
---------------
    try {
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    ...
}
```

跟踪以下这句代码：
```
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);

protected @Nullable Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
	if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
		// Make sure bean class is actually resolved at this point.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			Class<?> targetType = determineTargetType(beanName, mbd);
			if (targetType != null) {
				bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
				if (bean != null) {
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
		}
		mbd.beforeInstantiationResolved = (bean != null);
	}
	return bean;
}

protected @Nullable Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
		Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
		if (result != null) {
			return result;
		}
	}
	return null;
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessAfterInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```
Spring 会调用 `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(Class<?> beanClass, String beanName)` 尝试生成一个bean 实例，如果用户注册了自定义的 `InstantiationAwareBeanPostProcessor` 并且在 `postProcessBeforeInstantiation(...)` 方法中返回了一个实例对象，那么 Spring 就会使用这个实例返回，从代码中可以看出，如果有多个 `InstantiationAwareBeanPostProcessor` ，会依次调用，只要有一个返回了非 `null` 实例，就会使用这个实例对象，后续的 `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(...)` 方法不会继续调用。

另外，如果成功获取到实例对象后，也会 `BeanPostProcessor.postProcessAfterInitialization(...)` 方法处理这个实例对象。

综上所述，我们可以自定义 `InstantiationAwareBeanPostProcessor` 并在 `public @Nullable Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException` 方法中对特定 `Class` 类型的 bean 进行特殊处理，返回一个自定义的 bean 。

比如 Spring 自身的 `AbstractAutoProxyCreator` ：
```
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    ...
	@Override
	public @Nullable Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}
	...
}
```

OK，回到 `createBean(...)` ，这个方法的核心逻辑就是
1. 尝试 `Object bean = resolveBeforeInstantiation(beanName, mbdToUse)` 获取 bean，如果有定义的 `InstantiationAwareBeanPostProcessor` 返回了 bean 实例，则使用 `BeanPostProcessor`处理完bean后直接返回该 bean 实例
2. 否则继续执行 `doCreateBean(...)` 方法生成 bean

### doCreateBean-AbstractAutowireCapableBeanFactory
```
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object @Nullable [] args)
			throws BeanCreationException {
-----------
	// Instantiate the bean.
	// 实例化 bean，主要方法是 createBeanInstance(beanName, mbd, args);
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	Object bean = instanceWrapper.getWrappedInstance();
-----------
	// 调用 MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(...) 方法处理 RootBeanDefinition
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			...
			mbd.markAsPostProcessed();
		}
	}
-----------
	// 提前将 bean 放入缓存，为了解决循环依赖
	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		...
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		                             |
		                             |
		                             ▼
		//this.singletonFactories.put(beanName, singletonFactory);
		//this.earlySingletonObjects.remove(beanName);
		//this.registeredSingletons.add(beanName);
	}
-----------
	// 属性填充和初始化，依赖项的注入就发生在这里
	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		populateBean(beanName, mbd, instanceWrapper);
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	....
-----------
	// 检查 earlySingletonReferences 是否有 circular dependencies
	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = CollectionUtils.newLinkedHashSet(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(...);
				}
			}
		}
	}

-----------
	// 处理 DisposableBean 生命周期回调功能，会把bean 包装成 DisposableBean 并放到 disposableBeans Map 中。
	// ConfigurableBeanFactory.destroySingletons 的方法会在容器关闭的时候被调用，shutdown hook 时也会调用到
	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	....

	return exposedObject;
}
```

核心条线：
```
instanceWrapper = createBeanInstance(beanName, mbd, args); // 实例化bean
applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName); //

// 自动注入与初始化
populateBean(beanName, mbd, instanceWrapper);
exposedObject = initializeBean(beanName, exposedObject, mbd);

//其他循环引用问题处理及回调
```

#### createBeanInstance-AbstractAutowireCapableBeanFactory
```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object @Nullable [] args) {
	// Make sure bean class is actually resolved at this point.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);
	...

-----------
	// 从 RootBeanDefinition 获取 instanceSupplier，调用 get() 获取 bean 实例
	if (args == null) {
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName, mbd);
		}
	}
-----------
	//如果注册了工厂方法，使用工厂方法生成实例
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

-----------
	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			return instantiateBean(beanName, mbd);
		}
	}
-----------
	// Candidate constructors for autowiring?
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		return autowireConstructor(beanName, mbd, ctors, args);
	}
-----------
	// Preferred constructors for default construction?
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}
-----------
	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
```

#### instantiateBean-AbstractAutowireCapableBeanFactory
```
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

实例化的时候使用了策略模式， `InstantiationStrategy` 负责实例化 bean 。 Spring 提供了两种实例化策略： 
+ `SimpleInstantiationStrategy` 
+  `CglibSubclassingInstantiationStrategy`, `CglibSubclassingInstantiationStrategy` 继承自 `SimpleInstantiationStrategy`。
默认情况下使用的是 `CglibSubclassingInstantiationStrategy`。这点从 `AbstractAutowireCapableBeanFactory` 的构造函数可以看出：
```
public AbstractAutowireCapableBeanFactory() {
	super();
	ignoreDependencyInterface(BeanNameAware.class);
	ignoreDependencyInterface(BeanFactoryAware.class);
	ignoreDependencyInterface(BeanClassLoaderAware.class);
	this.instantiationStrategy = new CglibSubclassingInstantiationStrategy();
}
```

#### populateBean-AbstractAutowireCapableBeanFactory
```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	...
-----------
	// 调用 InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(...) 方法
	// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
	// state of the bean before properties are set. This can be used, for example,
	// to support styles of field injection.
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
			if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
				return;
			}
		}
	}
-----------
	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

	int resolvedAutowireMode = mbd.getResolvedAutowireMode();
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable.
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
		if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
-----------
	// 调用 InstantiationAwareBeanPostProcessor.postProcessProperties(...) 方法	
	if (hasInstantiationAwareBeanPostProcessors()) {
		if (pvs == null) {
			pvs = mbd.getPropertyValues();
		}
		for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
			PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
			if (pvsToUse == null) {
				return;
			}
			pvs = pvsToUse;
		}
	}
-----------
	boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
	if (needsDepCheck) {
		PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		checkDependencies(beanName, mbd, filteredPds, pvs);
	}

	if (pvs != null) {
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}
```

这个方法中有两处重点：
1. 是会依次调用 `InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(...)` 方法，并且只要有一个返回 `false`，就会直接返回，不会继续执行后续的属性填充和初始化操作。
2. 会依次调用 `InstantiationAwareBeanPostProcessor.postProcessProperties(...)` 方法，如果返回 `null`，就会直接返回，不会继续执行后续的属性填充和初始化操作。

重点看下第2点，Spring 的 `AutowiredAnnotationBeanPostProcessor` 就通过实现 `InstantiationAwareBeanPostProcessor` 接口，重写 `postProcessProperties(...)` 方法，来实现自动注入的功能的。
```
@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		....
		return pvs;
	}
```

#### initializeBean-AbstractAutowireCapableBeanFactory
```
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
	...
	invokeAwareMethods(beanName, bean); // 调用 Aware 相关的方法

	// 调用 BeanPostProcessor.postProcessBeforeInitialization(...) 方法
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		invokeInitMethods(beanName, wrappedBean, mbd); //调用 InitializingBean.afterPropertiesSet() 方法和自定义的初始化方法
	}
	...
	if (mbd == null || !mbd.isSynthetic()) {
		// 调用 BeanPostProcessor.postProcessAfterInitialization(...) 方法
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}

	return wrappedBean;
}
```
这里主要处理了3个接口相关的回调：
1. 实现 `Aware` 相关的回调
2. 调用 `BeanPostProcessor.postProcessBeforeInitialization(...)` 方法
3. 调用 `InitializingBean.afterPropertiesSet()` 方法和自定义的初始化方法
4. 调用 `BeanPostProcessor.postProcessAfterInitialization(...)` 方法

有一点需要注意的是 `ApplicationContextAwareProcessor` 是以 `BeanPostProcessor` 的形式来处理一些 `Aware` 相关的回调的。而 `invokeAwareMethods(...)` 则是固定地直接调用几个 `Aware` 相关的方法的。
```
private void invokeAwareMethods(String beanName, Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware beanNameAware) {
			beanNameAware.setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {
			ClassLoader bcl = getBeanClassLoader();
			if (bcl != null) {
				beanClassLoaderAware.setBeanClassLoader(bcl);
			}
		}
		if (bean instanceof BeanFactoryAware beanFactoryAware) {
			beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}

ApplicationContextAwareProcessor

public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
	...
	AccessControlContext acc = null;

	if (System.getSecurityManager() != null) {
		acc = this.applicationContext.getBeanFactory().getAccessControlContext();
	}

	if (acc != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			invokeAwareInterfaces(bean);
			return null;
		}, acc);
	}
	else {
		invokeAwareInterfaces(bean);
	}

	return bean;
}

private void invokeAwareInterfaces(Object bean) {
	if (bean instanceof EnvironmentAware environmentAware) {
		environmentAware.setEnvironment(this.applicationContext.getEnvironment());
	}
	if (bean instanceof EmbeddedValueResolverAware embeddedValueResolverAware) {
		embeddedValueResolverAware.setEmbeddedValueResolver(this.embeddedValueResolver);
	}
	if (bean instanceof ResourceLoaderAware resourceLoaderAware) {
		resourceLoaderAware.setResourceLoader(this.applicationContext);
	}
	if (bean instanceof ApplicationEventPublisherAware applicationEventPublisherAware) {
		applicationEventPublisherAware.setApplicationEventPublisher(this.applicationContext);
	}
	if (bean instanceof MessageSourceAware messageSourceAware) {
		messageSourceAware.setMessageSource(this.applicationContext);
	}
	if (bean instanceof ApplicationStartupAware applicationStartupAware) {
		applicationStartupAware.setApplicationStartup(this.applicationContext.getApplicationStartup());
	}
	if (bean instanceof ApplicationContextAware applicationContextAware) {
		applicationContextAware.setApplicationContext(this.applicationContext);
	}
}
```

## 循环依赖与三级缓存分析
+ `singletonObjects` : 保存已经创建好的 bean
+ `earlySingletonObjects` : 保存 early singleton 的 bean，early singleton 是指那些已经实例化，但是还没有完成属性填充和初始化的 bean
+ `singletonFactories` : 保存 bean 的 `ObjectFactory` ，用于创建 bean 实例

1. 当获取一个 bean 时，首先尝试从缓存 `singletonObjects` 中获取，如果获取不到，进入第2步
2. 尝试从 `earlySingletonObjects` 缓存中获取，如果获取不到，进入第3步
3. 从 `singletonFactories` 中获取，并将其放入 `earlySingletonObjects` 缓存中，然后从 `singletonFactories` 中移除
```
protected @Nullable Object getSingleton(String beanName, boolean allowEarlyReference) {
	// Quick check for existing instance without full singleton lock.
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		singletonObject = this.earlySingletonObjects.get(beanName);
		if (singletonObject == null && allowEarlyReference) {
			if (!this.singletonLock.tryLock()) {
				// Avoid early singleton inference outside of original creation thread.
				return null;
			}
			try {
				// Consistent creation of early reference within full singleton lock.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					singletonObject = this.earlySingletonObjects.get(beanName);
					if (singletonObject == null) {
						ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
						if (singletonFactory != null) {
							singletonObject = singletonFactory.getObject();
							// Singleton could have been added or removed in the meantime.
							if (this.singletonFactories.remove(beanName) != null) {
								this.earlySingletonObjects.put(beanName, singletonObject);
							}
							else {
								// singletonObject 已经被创建完成，从 singletonObjects 中获取
								singletonObject = this.singletonObjects.get(beanName);
							}
						}
					}
				}
			}
			finally {
				this.singletonLock.unlock();
			}
		}
	}
	return singletonObject;
}
```


而在 `AbstractAutowireCapableBeanFactory.doCreateBean(...)` 方法中， bean 实例创建好之后，进行属性依赖注入和初始化回调之前，如果允许循环依赖，则会将 bean 提前放入 `singletonFactories` 缓存中，为了解决循环依赖问题：

```
-----------
// 提前将 bean 放入缓存，为了解决循环依赖
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
		isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
	...
	addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
									|
									|
									▼
	//this.singletonFactories.put(beanName, singletonFactory);
	//this.earlySingletonObjects.remove(beanName);
	//this.registeredSingletons.add(beanName);
}
-----------
```

如果是构造函数循环依赖的情况，在实例化阶段就会去解析依赖，此时当前 bean 实例还未创建完成，也就无法将 bean 提前放入 `singletonFactories` 缓存中，所以构造函数循环依赖无法解决。

`A <--> B` ，A首先实例化好后，放进 `singletonFactories` 缓存，然后在 `populateBean` 方法时发现依赖 `B`，于是就创建 `B` 的 bean； `B` 实例化好之后，也放入到 `singletonFactories`，然后在 `populateBean` 方法时发现依赖 `A`，于是就尝试获取 `A` 的 bean，然后在 上述的 `getSingleton(...)` 方法中尝试从 `singletonFactories` 缓存中获取 `A` 的 bean，此时能够获取到 `A` 的 bean，然后 `B` 创建完成；最后 `A` 创建完成。