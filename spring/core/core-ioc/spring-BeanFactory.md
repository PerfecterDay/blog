# The BeanFactory API
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/beanfactory.html


`BeanFactory` API 为 Spring 的 IoC 功能提供了基础支撑。主要用于与 Spring 其他组件及相关第三方框架的集成，而其 `DefaultListableBeanFactory` 实现则是更高层级 `GenericApplicationContext` 的关键委托对象。

`BeanFactory` 及其相关接口（如 `BeanFactoryAware` 、`InitializingBean` 、 `DisposableBean` ）是其他框架组件的重要集成点。它们无需任何注解甚至反射机制，便能实现容器与其组件之间的高效交互。

核心 `BeanFactory` API 层及其 `DefaultListableBeanFactory` 实现并未对配置格式或任何组件注解的使用做出预设。所有这些特性均通过扩展（如 `XmlBeanDefinitionReader` 和 `AutowiredAnnotationBeanPostProcessor` ）实现，并基于共享的 `BeanDefinition` 对象作为核心元数据表示形式进行操作。这正是 Spring 容器如此灵活且可扩展的本质所在。

## BeanFactory or ApplicationContext?
除非有充分理由，否则应使用 `ApplicationContext` 。其中 `GenericApplicationContext` 及其子类 `AnnotationConfigApplicationContext` 是自定义引导的常用实现。这些是Spring核心容器的主要入口点，适用于所有常见场景：加载配置文件、触发类路径扫描、通过编程方式注册Bean定义和注解类，以及（从5.0版本起）注册功能性Bean定义。

由于 `ApplicationContext` 包含了 `BeanFactory` 的所有功能，因此通常推荐使用它而非普通的 `BeanFactory` ，除非需要完全控制bean处理过程。在 `ApplicationContext` （如 `GenericApplicationContext` 实现）内部，多种类型的Bean会通过约定方式被自动检测和注册（即通过Bean名称或Bean类型——特别各种 `postProcessor`），而普通的 `DefaultListableBeanFactory` 则对任何特殊Bean类型均不作特殊处理。

对于许多扩展容器功能（如注解处理和AOP代理）， `BeanPostProcessor` 扩展点至关重要。若仅使用普通的 `DefaultListableBeanFactory` ，此类后处理器默认不会被检测到并激活。

下表列出了 `BeanFactory` 与 `ApplicationContext` 的一些关键区别：

**Table 1. Feature Matrix**

| Feature | BeanFactory | ApplicationContext |
|--------|-------------|--------------------|
| Bean instantiation / wiring | Yes | Yes |
| Integrated lifecycle management | No | Yes |
| Automatic `BeanPostProcessor` registration | No | Yes |
| Automatic `BeanFactoryPostProcessor` registration | No | Yes |
| Convenient `MessageSource` access (for internationalization) | No | Yes |
| Built-in `ApplicationEvent` publication mechanism | No | Yes |


要显式地将一个 Bean 后处理器注册到 `DefaultListableBeanFactory` 中，需要通过编程方式调用 `addBeanPostProcessor` 方法，如下例所示：
```
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```

要将 `BeanFactoryPostProcessor` 应用于普通的 `DefaultListableBeanFactory` ，需要调用其 `postProcessBeanFactory` 方法，如下例所示：
```
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在两种情况下，显式注册步骤都显得不便，因此在Spring支持的应用程序中，各种 `ApplicationContext` 的实现比普通的 `DefaultListableBeanFactory` 更受青睐——尤其是在典型企业环境中，当需要依赖 `BeanFactoryPostProcessor` 和 `BeanPostProcessor` 实例来扩展容器功能时。

`AnnotationConfigApplicationContext` 注册了所有常见的注解后处理器，并可能通过配置注解（如 `@EnableTransactionManagement` ）在底层引入额外的处理器。在 Spring 基于注解的配置模型抽象层面上，bean post-processors 被隐藏为容器内部的细节。