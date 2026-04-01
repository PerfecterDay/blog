# Springboot 配置加载流程
{docsify-updated}

## 配置的读取与加载
```
SpringApplication.run(String... args)：
ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
|
SpringApplication.prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments)：
listeners.environmentPrepared(bootstrapContext, environment);
```

`EnvironmentPostProcessorApplicationListener` 监听到 `ApplicationEnvironmentPreparedEvent` 后会调用各种 `EnvironmentPostProcessor` 来处理配置加载到 `Environment` 实例中。常见的 `EnvironmentPostProcessor` 如下：

+ `BootstrapConfigFileApplicationListener`
+ `CachedRandomPropertySourceEnvironmentPostProcessor`
+ `CLoudFoundryVcapEnvironmentPostProcessor`
+ `ConfigDataEnvironmentPostProcessor`
+ `ConfigDataMissingEnvironmentPostProcessor`
+ `ConsulConfigDataMissingEnvironmentPostProcessor`
+ `DecryptEnvironmentPostProcessor`
+ `GatewayEnvironmentPostProcessor`
+ `HostInfoEnvironmentPostProcessor`
+ `IntegrationPropertiesEnvironmentPostProcessor`
+ `LogCorrelationEnvironmentPostProcessor`
+ `RandomValuePropertySourceEnvironmentPostProcessor`
+ `ReactorEnvironmentPostProcessor`
+ `SpringApplicationJsonEnvironmentPostProcessor`
+ `SystemEnvironmentPropertySourceEnvironmentPostProcessor`

<center><img src="pics/environmentPostprocessor.png" alt=""></center>

### PropertySource
`PropertySource` 是一个抽象基类，表示key/value 属性值对的源。底层源对象可以是封装 key/value 的任意类型T。比如底层封装 `java.util.Properties` 对象、`java.util.Map` 对象、 `ServletContext` 或 `ServletConfig` 对象（用于访问初始化参数）。 `PropertySource` 还有一个 `String name` 字段，代表每一个 `PropertySource` 对象都有一个关联的名字。 `PropertySource` 重要的方法如下：
```
public boolean containsProperty(String name) {
    return (getProperty(name) != null);
}

public T getSource() {
    return this.source;
}

public String getName() {
    return this.name;
}

public abstract Object getProperty(String name);
```

### PropertySourceLoader
加载配置文件或者其他地方的配置资源为 `PropertySource` 对象 . 主要实现者：
+ `PropertiesPropertySourceLoader` : 读取 properties 文件
+ `YamlPropertySourceLoader` : 加载 yml 文件


## 配置值与 bean 属性绑定
Spring Framework（以及 Spring Boot）通过一套精密的机制将 `Environment` 中 `PropertySource` 里的配置值绑定到 Java Bean 对象的字段上。这个过程主要涉及以下三个核心组件和机制：

### `@Value` 
这是最直接、最基础的绑定方式。  
实现者： `AutowiredAnnotationBeanPostProcessor` 和 `ValueAnnotationBeanPostProcessor` 。  
工作原理：  
当 Spring 容器实例化一个 Bean 之后（Post-Processing 阶段）。 BeanPostProcessor 会检查该 Bean 的字段或方法参数上是否有 `@Value("${property.key}")` 注解。它使用 Spring 的 表达式语言（SpEL） 解析 "${...}" 占位符。解析 SpEL 时，它会调用 `Environment.getProperty("property.key")` 从当前的 Environment 中获取对应的值。  
最后，它将获取到的值注入到 Bean 的字段中。  
局限性： 只能用于单个属性的注入，需要为每个属性单独标注。

### `@ConfigurationProperties` 
对于需要绑定大量相关配置的场景，Spring Boot 提供了更高级、更强大的 `ConfigurationProperties` 绑定机制。

实现者： `ConfigurationPropertiesBindingPostProcessor` （由 Spring Boot 提供）。

工作原理：  
你创建一个 POJO 类（配置 Bean），并在类上使用 `@ConfigurationProperties(prefix = "app.settings")`。  
`ConfigurationPropertiesBindingPostProcessor` 在 Bean 实例化后介入。  
它使用 Spring Boot 的 `Binder` API。  
`Binder` 会以 `prefix` （例如 app.settings）为起点，扫描 `Environment` 中的所有 `PropertySource` 。  
它查找所有以 `app.settings.` 开头的属性（例如 `app.settings.timeout` 或 `app.settings.port` ）。  
`Binder` 采用灵活的命名规则（如宽松绑定，`kebab-case` 到 `camelCase` 转换），将找到的值自动匹配并设置到配置 Bean 的对应字段上。

优势： 支持类型转换、验证（通过 `@Validated`）、复杂嵌套对象、列表、Map 等，是 Spring Boot 推荐的配置绑定方式。

### 类型转换
无论使用 `@Value` 还是 `@ConfigurationProperties` ，从 `PropertySource` 中读取的值通常都是 `String` 类型。Spring 必须将其转换为 Bean 字段的实际类型（如 `Integer` 、`Duration`、自定义枚举等）。

实现者： `ConversionService。`

工作原理： 在绑定过程中，Spring 使用 `ConversionService` 来执行类型转换。Spring Boot 默认注册了大量的转换器，可以自动处理常见的类型转换。如果你使用了自定义类型，则需要注册自己的 `Converter` 或 `Formatter` 。

### 总结
整个配置绑定流程大致如下：
1. 启动阶段： `EnvironmentPostProcessor` （例如加载 application.yml 的组件）将配置文件的内容解析成 `PropertySource` ，并添加到 `Environment` 中。
2. Bean 创建阶段： Spring 容器实例化 Bean。
3. 后置处理阶段：
   + 对于 `@Value`： `BeanPostProcessor` 使用 SpEL 调用 `Environment.getProperty(key)` 获取值，并注入。
   + 对于 `@ConfigurationProperties` ： `ConfigurationPropertiesBindingPostProcessor` 使用 `Binder` API 扫描 `Environment` 中特定前缀下的所有属性，执行类型转换和宽松绑定，然后将值设置到配置 Bean 的字段上。
