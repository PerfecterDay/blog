# Spring启动原理-SpringApplication实例化
{docsify-updated}

绝大部分Springboot 的启动部分都是:

```java
public static void main(String[] args) throws Exception {
	SpringApplication.run(xxx.class, args);
}
```

`SpringApplication` 的静态 `run` 方法首先会创建一个 SpringApplication 对象并调用它的 run 方法: 

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

SpringApplication 对象的构造过程入下：

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

实例化过程中多次调用了 `getSpringFactoriesInstances(Class<T> type)` 方法，初始化 SpringApplication 对象的一些属性：

<center><img src="pics/spring-application.png" width="60%"></center>

此处使用了 Springboot 的 SPI 机制 `SpringFactoriesLoader` 。

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

### SpringFactoriesLoader 
以下是Springboot源码中对 SpringFactoriesLoader 的注释：

> Load and instantiate the factory implementations of the given type from "META-INF/spring.factories", using the configured class loader, the given argument resolver, and custom failure handling provided by the given failure handler.  
> The returned factories are sorted through AnnotationAwareOrderComparator.As of Spring Framework 5.3, if duplicate implementation class names are discovered for a given factory type, only one instance of the duplicated implementation type will be instantiated.  
> For any factory implementation class that cannot be loaded or error that occurs while instantiating it, the given failure handler is called.

```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"; //定义了读取文件的位置

public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    result = new HashMap<>();
    try {
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] factoryImplementationNames =
                        StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                for (String factoryImplementationName : factoryImplementationNames) {
                    result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                            .add(factoryImplementationName.trim());
                }
            }
        }

        // Replace all lists with unmodifiable lists containing unique elements
        result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
        cache.put(classLoader, result);
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
}
```

### BootstrapRegistryInitializer
`SpringApplication` 实例化的时候会使用 SPI 初始化加载 `BootstrapRegistryInitializer` 、 `ApplicationContextInitializer` 和 `ApplicationListener` 实例。其中 `BootstrapRegistryInitializer` 用于初始化 `BootstrapContext` 。

```java
private DefaultBootstrapContext createBootstrapContext() {
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
    return bootstrapContext;
}
```

那么 `BootstrapContext` 有什么用了？看官方注释：

A simple bootstrap context that is available during startup and {@link Environment}
post-processing up to the point that the {@link ApplicationContext} is prepared.
Provides lazy access to singletons that may be expensive to create, or need to be
shared before the {@link ApplicationContext} is available.

<center><img src="pics/bootstrapcontext.png" width="60%"></center>

从注释及源码中可以看出 `BootstrapContext` 主要在 `ApplicationContext` 准备好之前充当容器功能使用。

### ApplicationContextInitializer和ApplicationListener
`ApplicationContextInitializer` 和 `ApplicationListener` 在后文会介绍。下边是 SpringApplication 实例化后加载到的实例：

<center><img src="/pics/springApplication.png" alt=""></center>
