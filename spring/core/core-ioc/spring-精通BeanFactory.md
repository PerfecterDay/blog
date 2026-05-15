# 一文精通 Spring BeanFactory
{docsify-updated}

## BeanDefinition
描述一个 bean 的元数据信息，比如 `class` 类型 , `scope` , `name` , `isLazyInit` 等信息。
```
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {...}

public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {...}

public class RootBeanDefinition extends AbstractBeanDefinition {...}

public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {...}

public class ScannedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {...}
```

## BeanDefinitionRegistry
注册 `BeanDefinition` 元信息的接口：
```
public interface BeanDefinitionRegistry extends AliasRegistry {
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
    ...
}
```

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

### AbstractBeanFactory.doGetBean
```
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
    ...
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //保存创建好的 singleton bean 
    private final Set<String> singletonsCurrentlyInCreation = ConcurrentHashMap.newKeySet(16);
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16); //Cache of early singleton objects: bean name to bean instance.
	private final Set<String> alreadyCreated = ConcurrentHashMap.newKeySet(256);

    private final List<BeanPostProcessor> beanPostProcessors = new BeanPostProcessorCacheAwareList();
    private final Map<String, Scope> scopes = new LinkedHashMap<>(8);
    private ApplicationStartup applicationStartup = ApplicationStartup.DEFAULT;
    private final Map<String, RootBeanDefinition> mergedBeanDefinitions = new ConcurrentHashMap<>(256); //最终存放 BeanDefinition 元信息
    private final Set<String> alreadyCreated = ConcurrentHashMap.newKeySet(256);
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
        // 可以手动在 BeanFactory 中注册一个bean，这种情况就走下边这个分支
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			...
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
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
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
						catch (BeanCreationException ex) {
							if (requiredType != null) {
								// Wrap exception with current bean metadata but only if specifically
								// requested (indicated by required type), not for depends-on cascades.
								throw new BeanCreationException(mbd.getResourceDescription(), beanName,
										"Failed to initialize dependency '" + ex.getBeanName() + "' of " +
												requiredType.getSimpleName() + " bean '" + beanName + "': " +
												ex.getMessage(), ex);
							}
							throw ex;
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
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
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
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
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
					catch (IllegalStateException ex) {
						throw new ScopeNotActiveException(beanName, scopeName, ex);
					}
				}
			}

			catch (BeansException ex) {
				beanCreation.tag("exception", ex.getClass().toString());
				beanCreation.tag("message", String.valueOf(ex.getMessage()));
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
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

最终都会调用 `createBean(...)` 方法。

### AbstractAutowireCapableBeanFactory.createBean
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

OK，回到 `createBean(...)` ，如果 `Object bean = resolveBeforeInstantiation(beanName, mbdToUse)` 返回的是 `null`，则会继续执行 `doCreateBean(...)` 方法，否则会直接返回这个实例对象。

### AbstractAutowireCapableBeanFactory.doCreateBean
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
	// 处理 DisposableBean 生命周期回调功能
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

#### AbstractAutowireCapableBeanFactory.createBeanInstance
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

#### AbstractAutowireCapableBeanFactory.instantiateBean
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

#### AbstractAutowireCapableBeanFactory.populateBean
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

#### AbstractAutowireCapableBeanFactory.initializeBean
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
1. `Aware` 相关的回调
2. `BeanPostProcessor.postProcessBeforeInitialization(...)` 方法
3. `InitializingBean.afterPropertiesSet()` 方法和自定义的初始化方法
4. `BeanPostProcessor.postProcessAfterInitialization(...)` 方法

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

## 总结
Spring 的 `BeanFactory` 中获取 bean 时，有几个扩展点：
1. 如果自己在 `BeanFactory` 中注册了 bean，直接返回这个 bean，通过 `SingletonBeanRegistry.registerSingleton(String beanName, Object singletonObject)` 方法能注册
2. 如果自定义了 `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(...)` 方法，并且返回了 bean 实例，则直接返回这个 bean 实例。这种情况下，如果成功获取到实例对象后，也会 `BeanPostProcessor.postProcessAfterInitialization(...)` 方法处理这个实例对象。
3. 如果注册 bean 时提供了 `Supplier` ，则会使用 `Supplier` 生成实例
4. 使用 `InstantiationStrategy` 实例化策略实例化 bean
5. `InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(...)` 方法处理实例化后的 bean，这时候 bean 中的依赖、初始化回调、 `Aware` 相关的回调都还没有执行。
6. `InstantiationAwareBeanPostProcessor.postProcessProperties(...)` 方法处理 bean 的属性，可以注入、修改、添加、删除 bean 的属性值。
7. 当 bean 实例化并且完成属性的注入后，在`InitializingBean.afterPropertiesSet()`/自定义初始化方法执行之前会调用 `BeanPostProcessor.postProcessBeforeInitialization(...)` 方法处理 bean。
8. 执行完初始化方法之后会执行 `BeanPostProcessor.postProcessAfterInitialization(...)` 方法处理 bean。

1，2，3，4 处理的都是对象的实例化过程，5，6 处理的都是对象实例化后，属性注入前的处理，7，8 处理的都是对象实例化并且属性注入后，初始化前后的处理。
