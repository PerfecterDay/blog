# Spring 杂谈
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

### AbstractBeanFactory
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

核心的 doGetBean 方法：
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

### AbstractAutowireCapableBeanFactory
```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object @Nullable [] args)
        throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
        try {
            mbdToUse.prepareMethodOverrides();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                    beanName, "Validation of method overrides failed", ex);
        }
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

SmartInstantiationAwareBeanPostProcessor
InstantiationAwareBeanPostProcessor


DestructionAwareBeanPostProcessor
MergedBeanDefinitionPostProcessor


Supplier instanceSupplier

SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors(...)

InstantiationStrategy/SimpleInstantiationStrategy/CglibSubclassingInstantiationStrategy



BeanWrapper/BeanWrapperImpl


spring 中实际使用的是 `DefaultListableBeanFactory`， `DefaultListableBeanFactory` 使用 `ConcurrentHashMap` 来保存 `BeanDefinition` 元信息。
```
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
    ...
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    ...
}
```