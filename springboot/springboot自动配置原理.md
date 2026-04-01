#  Springboot 自动配置原理
{docsify-updated}

大体的步骤是：
1. springboot 官方提供了很多有自动配置功能的配置类。这些配置类如何被扫描加载呢？
2. `AutoConfigurationImportSelector` 这个类会扫描特定目录下的文件来加载一些配置类或者接口实现类，springboot官方的自动配置类就在这些扫描目录下
3. `@EnableAutoConfiguration` 使用 `@Import(AutoConfigurationImportSelector)` 导入了 `AutoConfigurationImportSelector` 从而会导入自动配置类，`@EnableAutoConfiguration` 又由 `@SpringBootApplication` 导入。为什么`@Import` 能实现这个导入功能呢？由 `ConfigurationClassParser` 类实现一些注解的导入功能
4. `ConfigurationClassParser` 又是如何被引入的呢？ `ConfigurationClassPostProcessor` 在加载读取配置类的时候会实例化 `ConfigurationClassParser`
5. `ConfigurationClassPostProcessor` 本身在 springboot 创建 applicationContext 会在容器中注册 `ConfigurationClassPostProcessor` 类型的 bean
6. `ConfigurationClassPostProcessor <-- BeanDefinitionRegistryPostProcessor <-- BeanFactoryPostProcessor`，所以会在容器启动阶段被调用（ `invokeBeanFactoryPostProcessors(beanFactory)` ）

## 注解扫描的实现机制
自动配置的本质还是Spring扫描配置类/配置文件，加载基于注解或者xml配置的 bean。那么首先，依然是要扫描配置类。

Spring基于注解配置功能主要依赖 `BeanPostProcessor/BeanFactoryPostProcessor` 扩展来实现的，一些典型的 `BeanPostProcessor/BeanFactoryPostProcessor` 如下：
+ `ConfigurationClassPostProcessor` 
+ `AutowiredAnnotationBeanPostProcessor`
+ `CommonAnnotationBeanPostProcessor`
+ `PersistenceAnnotationBeanPostProcessor`
+ `EventListenerMethodProcessor`

其中某些 `xxxProcessor` 的加载过程如下：
<center><img src="pics/create_context.svg" width=""></center>

所以springboot在启动时就会自动注册这些 BeafnDefinition,加载了这些 `xxxProcessor` 后，就能扫描配置类了。自动配置类与普通配置类有一些些差别。

## 自动配置类
Springboot（通常是各种starter） 的自动配置实际上就是为我们写好了很多 `@Configuration` 注解的配置类，这些类中大量使用了基于 `@Conditional` 注解的配置，以在满足一些条件时自动为我们注入一些 Bean 。那么还有一个问题，我们知道，要使 `@Configuration` 注解的配置类生效，主要有三种方式：
1. 它处于自动扫描的包下，会被自动扫描(上文讲述了)
2. 被其他配置类用 `@Import` 引用
3. 采用编程的方式手动注册到应用上下文
   ```
   AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
   context.register(ExampleConfiguration.class);
   ```
这些方式都必须要求使用者（写代码）手写相关的代码才能完成。

Springboot 是如何加载这些配置类的呢？

## @Import 注解
+ 导入组件：@Import 注解允许你导入一个或多个组件类，通常是 @Configuration 配置类，到你的 Spring 应用上下文中。
+ 功能性：它提供了与 Spring XML 配置中的 <import/> 元素类似的功能，使配置模块化和易于管理。
+ 用法：你可以使用它导入：
+ `@Configuration` 配置类，这些类定义了 bean 和配置设置。
+ `ImportSelector` 实现，可以根据某些条件有选择地导入配置。
+ `ImportBeanDefinitionRegistrar` 实现，可以通过编程方式注册 beanDefinition 定义。

下图展示了Spring/Springboot 是如何处理 `@Import` 注解的：
<center><img src="pics/import.svg" alt=""></center>


**`ConfigurationClassParser.doProcessConfigurationClass(...)` 方法中可以看到很多处理各种注解（`@PropertySource/@ComponentScan/@Import/@ImportResource/@Bean`）的代码。如果想了解Springboot中各个注解是如何工作的，可以参照这个类中的代码。**

## Springboot自动配置原理

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ....
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    ....
}
```

<center><img src="pics/process-imports.png" width="60%"></center>

通过 `@SpringBootApplication`注解， `AutoConfigurationImportSelector` 和 `AutoConfigurationPackages.Registrar` 被 `@Import` 导入。

Springboot 中的`AutoConfigurationImportSelector`（委托 `ImportCandidates` 真正实现）会检查发布的 jar 中的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。

<center><img src="pics/auto-config.png" alt=""></center>

该文件应列出配置类，每行一个类名，如下例所示：
```
com.mycorp.libx.autoconfigure.LibXAutoConfiguration
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
```
这些类中往往使用了 `@Conditional` 等注解实现了自定义的自动配置。

自动配置必须通过在导入文件中命名的方式加载。确保它们被定义在特定的包空间中，并且永远不会成为组件扫描的目标。此外，自动配置类不应允许组件扫描查找其他组件,应使用特定的 `@Import` 注解来代替。

## @Configuration 注解
使用 @Configuration 注解的类，可以在其内定义用 @Bean 注解的方法，方法的返回对象将注入到 Spring bean 容器中 。

## @Conditional 注解
`@Conditional` 是最基础的注解，许多其他注解都是扩展自该注解。

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
    Class<? extends Condition>[] value();
}
```
该注解的值是一个继承自 `Condition` 接口的类， `Condition` 接口只有一个返回 `boolean` 的 `matches` 方法：
```
@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
所以该注解可以和`@Conmponent/@Service/@Repository/@Configuration`等一起标注在一个类上，当Condition中的接口返回 true 时，这些注解才会生效。另外，该注解也可以和 `@Bean` 注解一起使用，以控制一个 bean 只在条件满足的时候才会被注入。

基于 `@Conditional` 注解， springboot 实现了很多扩展的注解，这些注解通常配合 `@Configuration` 注解一起使用以控制某个配置类是否生效，来达到自动配置的效果：
1. `@ConditionalOnClass/@ConditionalOnMissingClass` : 当classpath中存在/不存在某个类时，配置生效
2. `@ConditionalOnMissingBean/@ConditionalOnBean` ：当spring上下文/beanfactory 中缺失/存在某个类型bean 时，配置生效
3. `@ConditionalOnNotWebApplication/@ConditionalOnWebApplication`: 只有时Web 类型应用时配置生效
4. `@ConditionalOnProperty("my.property")`：只有配置项 my.property 存在时，配置才生效
5. `@ConditionalOnResource("classpath:my.properties")`：只有配置文件 my.property 存在时，配置生效
6. `@ConditionalOnJava(JavaVersion.EIGHT)`：只有Java 版本时才生效
7. `@ConditionalOnCloudPlatform(CloudPlatform.Heroku)`：只有在指定的云平台下才生效
8. `@ConditionalOnExpression("someSpELExpression)`：只有在SPEL表达式为true 的情况下生效
9. `@ConditionalOnJndi("java:comp/env/ejb/myEJB")`：只有在指定jndi存在的情况下才生效

更多 `@Conditional` 注解，可参考[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.condition-annotations)

## 自定义自动配置  [官方指导](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)
自动配置必须通过在导入文件中命名的方式加载。确保它们被定义在特定的包空间中，并且永远不会成为组件扫描的目标。此外，自动配置类不应允许组件扫描查找其他组件。应使用特定的 `@Import` 注解来代替。

另外，自动配置类必须啊放在一个单独的jar 中。

1. 新建一个自动配置类
   ```
    @Configuration
    public class AutoConfigurationTest {

        @Bean
        @ConditionalOnProperty("test")
        public Test test(){
            return new Test();
        }
    }
   ```
2. resources 目录下新建 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件并写入以下内容。
   `com.demo.test.AutoConfigurationTest`

## 自定义starter [官方指导](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.custom-starter)
Spring Boot 2.7 引入了用于注册自动配置的新 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，同时保持了与 spring.factories 中注册的向后兼容性。在此版本中，使用 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 关键字在 `spring.factories` 中注册自动配置的支持已被移除，转而使用 imports 文件。`spring.factories` 中其他键下的条目不受影响。

Spring Boot 会检查发布的 jar 中是否存在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。该文件应列出配置类，每行一个类名，如下例所示：
```
com.mycorp.libx.autoconfigure.LibXAutoConfiguration
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
```

### @SpringBootApplication 注解
通常我们使用 @SpringBootApplication 在启动 springboot 的 main 方法所在类上标注来启动自动注解功能。

`@SpringBootApplication` 实际上是三个注解的合集：
```
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication{...}
```

1. @SpringBootConfiguration  

    `@SpringBootConfiguration` 是一个类级注解，是 Spring Boot 框架的一部分。它表示一个类提供了应用程序配置。
    `@SpringBootConfiguration` 是 `@Configuration` 注解的替代品。其主要区别在于 `@SpringBootConfiguration` 允许自动定位配置。这对单元测试或集成测试特别有用。
2. @EnableAutoConfiguration
  
   `@EnableAutoConfiguration` 注解使 Spring Boot 能够自动配置应用程序上下文。因此，它会根据类路径中包含的 jar 文件和我们定义的 Bean 自动创建和注册 Bean。
3. @ComponentScan

   @ComponentScan 使 Spring 能够扫描配置、控制器、服务和我们定义的其他组件。可以从指定的包开始扫描，使用 `basePackageClasses()` 或 `basePackages()` 来定义指定的包。如果没有指定包，Spring 会将声明 `@ComponentScan` 注解的类的所在的包作为起始包，扫描该包及其子包下的所有配置。