# 一文精通 Spring BeanDefinition 注册
{docsify-updated}

前边两篇文章我们详细介绍了 `BeanFactory` 与 `ApplicationContext` 的实现，本篇文章我们来详细介绍一下 Spring 扫描注册 `BeanDefinition` 的过程。

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

## BeanDefinitionBuilder
Programmatic means of constructing BeanDefinitions using the builder pattern. Intended primarily for use when implementing Spring 2.0 NamespaceHandlers.

## ClassPathScanningCandidateComponentProvider
```
LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
ClassPathScanningCandidateComponentProvider scanner = getScanner();
scanner.setResourceLoader(this.resourceLoader);
scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
Set<String> basePackages = getBasePackages(metadata);
Set<String> basePackages = getBasePackages(metadata);
for (String basePackage : basePackages) {
    candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
}
```

## BeanDefinitionRegistry
保存、注册、获取等管理 `BeanDefinition` 元信息的接口：
```
public interface BeanDefinitionRegistry extends AliasRegistry {
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
    ...
}
```

## ApplicationContext 的构造
Springboot 默认创建的 `ApplicationContext` 是 `AnnotationConfigServletWebServerApplicationContext` .
`AnnotationConfigApplicationContext`/`AnnotationConfigServletWebServerApplicationContext` 的构造函数中都会创建 `AnnotatedBeanDefinitionReader/ClassPathBeanDefinitionScanner` 来读取 `BeanDefinition` 。并且都会在构造函数中直接 `new` 这些对象：
```
AnnotationConfigApplicationContext
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

AnnotationConfigServletWebServerApplicationContext
public AnnotationConfigServletWebServerApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

## AnnotatedBeanDefinitionReader
以下是 `AnnotatedBeanDefinitionReader` 的构造函数：
```
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    this.registry = registry;
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

构造函数中主要实现了两个功能：
1. 实例化了 `ConditionEvaluator` 对象，该对象用于评估 `@Conditional` 注解的条件是否满足。
2. 调用了 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 方法，该方法会向 `BeanDefinitionRegistry` 中注册一些 `BeanFactoryPostProcessor` 、`BeanPostProcessor` 等。

`AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 方法如下：
```
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
        BeanDefinitionRegistry registry, @Nullable Object source) {

    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = CollectionUtils.newLinkedHashSet(6);

    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for Jakarta Annotations support, and if present add the CommonAnnotationBeanPostProcessor.
    if (JAKARTA_ANNOTATIONS_PRESENT && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (JPA_PRESENT && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                    AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
```
总结下来， `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 方法会视情况向 `BeanDefinitionRegistry` 中注册以下 `BeanDefinition` ：
1. `org.springframework.context.annotation.internalConfigurationAnnotationProcessor` -> `ConfigurationClassPostProcessor` : 处理 `@Configuration` 、`@Bean` 、`@Import` 、`@ImportResource` 、`@ComponentScan` 等注解，另外会将 `@PropertySource` 注解的属性文件加载到 `Environment` 中。
2. `org.springframework.context.annotation.internalAutowiredAnnotationProcessor` -> `AutowiredAnnotationBeanPostProcessor` : 处理 `@Autowired` 、`@Value` 、 `@Injec` 等注解
3. `org.springframework.context.annotation.internalCommonAnnotationProcessor` -> `CommonAnnotationBeanPostProcessor` : 处理 `@Resource` 、`@PostConstruct` 、`@PreDestroy` 等注解
4. `org.springframework.context.annotation.internalPersistenceAnnotationProcessor` -> `PersistenceAnnotationBeanPostProcessor` : 处理 JPA 相关注解，如果添加了 JPA 依赖
5. `org.springframework.context.event.internalEventListenerProcessor` -> `EventListenerMethodProcessor` : 处理 `@EventListener` 注解
6. `org.springframework.context.event.internalEventListenerFactory` -> `DefaultEventListenerFactory` : 事件监听器工厂

如下图，就是在 `AbstractApplicationContext` 的 `refresh()` 开始执行时，注册的 `BeanDefinition` ：

<center><img src="pics/annotationConfigUtils.png" width="80%"></center>

+ `ConfigurationClassPostProcessor` ，是一个 用于处理 `@Configuration` 注解类的 `BeanFactoryPostProcessor` 。
+ `AutowiredAnnotationBeanPostProcessor` ，是一个 `BeanPostProcessor` ，用于处理 `@Autowired` 、`@Value` 、 `@Injec` 等注解
+ `CommonAnnotationBeanPostProcessor` ，是一个 `BeanPostProcessor` ，用于处理 `@Resource` 、`@PostConstruct` 、`@PreDestroy` 等注解
+ `PersistenceAnnotationBeanPostProcessor` ，是一个 `BeanPostProcessor` ，用于处理 JPA 相关注解，如果添加了 JPA 依赖


## ConfigurationClassPostProcessor

```
ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
    └─ processConfigBeanDefinitions()
        ├─ ConfigurationClassParser.parse()             ← 解析阶段
        │   └─ processImports()
        │       └─ 遇到 ImportBeanDefinitionRegistrar：
        │           ├─ new Registrar() + 注入 Aware
        │           └─ configClass.addImportBeanDefinitionRegistrar(...)   ← 暂存
        │
        └─ ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()   ← 注册阶段
            └─ loadBeanDefinitionsForConfigurationClass(每个 configClass)
                └─ loadBeanDefinitionsFromRegistrars(...)
                    └─ registrar.registerBeanDefinitions(metadata, registry, ...)   ← 👈 调你的代码
```


### ConfigurationClassParser
```
protected final @Nullable SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {

    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass, sourceClass, filter);
    }

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), org.springframework.context.annotation.PropertySource.class,
            PropertySources.class, true)) {
        if (this.propertySourceRegistry != null) {
            this.propertySourceRegistry.processPropertySource(propertySource);
        }
       .....
    }

    // Search for locally declared @ComponentScan annotations first.
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScan.class, ComponentScans.class,
            MergedAnnotation::isDirectlyPresent);

    // Fall back to searching for @ComponentScan meta-annotations (which indirectly
    // includes locally declared composed annotations).
    if (componentScans.isEmpty()) {
        componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(),
                ComponentScan.class, ComponentScans.class, MergedAnnotation::isMetaPresent);
    }

    if (!componentScans.isEmpty()) {
        List<Condition> registerBeanConditions = collectRegisterBeanConditions(configClass);
        if (!registerBeanConditions.isEmpty()) {
            .....
        }
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

    // Process any @ImportResource annotations
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        if (methodMetadata.isAnnotated("kotlin.jvm.JvmStatic") && !methodMetadata.isStatic()) {
            continue;
        }
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java")) {
            boolean superclassKnown = this.knownSuperclasses.containsKey(superclass);
            this.knownSuperclasses.add(superclass, configClass);
            if (!superclassKnown) {
                // Superclass found, return its annotation metadata and recurse
                return sourceClass.getSuperClass();
            }
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

1. 处理 `@PropertySource` 注解
2. 处理 `@ComponentScan` 注解
3. `processImports(...)` 处理 `@Import` 注解
4. 处理 `@ImportResource` 注解
5. 处理 `@Bean` 注解


### @Import 注解
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

	/**
	 * {@link Configuration @Configuration}, {@link ImportSelector},
	 * {@link ImportBeanDefinitionRegistrar}, {@link BeanRegistrar}, or regular
	 * component classes to import.
	 */
	Class<?>[] value();

}
```

`ConfigurationClassParser` 中处理 `@Import` 的代码如下：
```
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, Predicate<String> filter, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                            this.environment, this.resourceLoader, this.registry);
                    Predicate<String> selectorFilter = selector.getExclusionFilter();
                    if (selectorFilter != null) {
                        filter = filter.or(selectorFilter);
                    }
                    if (selector instanceof DeferredImportSelector deferredImportSelector) {
                        this.deferredImportSelectorHandler.handle(configClass, deferredImportSelector);
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, filter);
                        processImports(configClass, currentSourceClass, importSourceClasses, filter, false);
                    }
                }
                else if (candidate.isAssignable(BeanRegistrar.class)) {
                    Class<?> candidateClass = candidate.loadClass();
                    BeanRegistrar registrar = (BeanRegistrar) BeanUtils.instantiateClass(candidateClass);
                    AnnotationMetadata metadata = currentSourceClass.getMetadata();
                    if (registrar instanceof ImportAware importAware) {
                        importAware.setImportMetadata(metadata);
                    }
                    configClass.addBeanRegistrar(metadata.getClassName(), registrar);
                }
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                    this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass), filter);
                }
            }
        }
        ....
        finally {
            this.importStack.pop();
        }
    }
}
```

以上代码可以看出，分别处理了 `ImportSelector` 、`BeanRegistrar` 、`ImportBeanDefinitionRegistrar` 、`@Configuration` 配置类。

### ImportBeanDefinitionRegistrar 的实例
```
@EnableAspectJAutoProxy → AspectJAutoProxyRegistrar
@EnableTransactionManagement → TransactionManagementConfigurationSelector（用 ImportSelector，但底层结合 Registrar）
@EnableFeignClients → FeignClientsRegistrar
@EnableConfigurationProperties → EnableConfigurationPropertiesRegistrar
Spring Data 的 @Enable*Repositories → *RepositoriesRegistrar
```

1. 它是单例 + 无状态的临时对象： `Registrar` 由 `new` 实例化，不进入 Spring 容器，用完就丢。所以不要在 `Registrar` 字段里存运行期状态。
2. 它能拿到 4 种"基础设施"，但拿不到完整 IoC
    通过两种方式注入：
    + 构造器注入：`public MyRegistrar(Environment env, BeanFactory bf, ClassLoader cl, ResourceLoader rl)` 任意组合
    + `Aware` 接口： `EnvironmentAware / BeanFactoryAware / BeanClassLoaderAware / ResourceLoaderAware`
    但拿不到其他 Bean 的实例（因为此时 Bean 都还没实例化）。
3. 不能在里面注册 `BeanDefinitionRegistryPostProcessor`
4. `Registrar` 的执行时机: 它在 `BeanFactoryPostProcessor` 阶段（具体说在 `BeanDefinitionRegistryPostProcessor` 阶段）执行，早于任何 Bean 的实例化。它注册的 `BeanDefinition` 也会被后续的 `BeanFactoryPostProcessor` （比如属性占位符解析）正常处理。


### ConfigurationClassParser 总结
`ConfigurationClassParser` 只是解析 `@Configuration` 标注的配置类，将其解析成 `ConfigurationClass` 中各个字段的配置：
```
final class ConfigurationClass {

	private final AnnotationMetadata metadata;

	private final Resource resource;

	@Nullable
	private String beanName;

	private boolean scanned = false;

	private final Set<ConfigurationClass> importedBy = new LinkedHashSet<>(1);

	private final Set<BeanMethod> beanMethods = new LinkedHashSet<>(); // @Bean 标注的方法

	private final Map<String, Class<? extends BeanDefinitionReader>> importedResources =
			new LinkedHashMap<>();

	private final Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> importBeanDefinitionRegistrars =
			new LinkedHashMap<>(); // @Import 导入的 ImportBeanDefinitionRegistrar

	final Set<String> skippedBeanMethods = new HashSet<>();
    ....
}
```
真正从配置类中加载 `BeanDefinition` 到 `BeanFactory` 中的是 `ConfigurationClassBeanDefinitionReader` 。

### ConfigurationClassBeanDefinitionReader
`ConfigurationClassBeanDefinitionReader` 负责根据 `ConfigurationClass` 中的配置，加载 `BeanDefinition` 到 `BeanFactory` 中。

```
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    for (ConfigurationClass configClass : configurationModel) {
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}

private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```