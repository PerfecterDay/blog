# Spring 的启动与配置
{docsify-updated}

- [Spring 的启动与配置](#spring-的启动与配置)
  - [Spring Framework 的启动](#spring-framework-的启动)
    - [JavaEE 环境下的启动](#javaee-环境下的启动)
      - [部署描述符启动](#部署描述符启动)
      - [初始化器中使用编程的方式启动 Spring](#初始化器中使用编程的方式启动-spring)
  - [Spring Framework 的配置方式](#spring-framework-的配置方式)
    - [创建 XML 配置](#创建-xml-配置)
    - [创建混合配置](#创建混合配置)
      - [组件扫描的开启](#组件扫描的开启)
      - [扫描注解配置](#扫描注解配置)
    - [使用 @Configuration 配置](#使用-configuration-配置)


## Spring Framework 的启动
Spring Framework 是另一个容器，它可以运行任何 Java SE和 Java EE  容器中，并作为应用程序的运行时环境。另外，如同计算机或者jvm的那个样例一样， Spring 必须被启动并且需要得到如何运行它所包含的应用程序的指令。
配置和启动 Spring Framework 是两个不同的任务，并且相互独立，都可以通过多种方式实现。配置告诉 Spring 如何运行它所包含的应用程序时，启动进程将启动 Spring Framework并将配置指令传递给它。

+ **在 Java SE 中，只有一种方式启动 Spring Framework：通过应用程序的 main 方法以编程的方式启动。**
+ **在 Java EE 应用程序中，有两种选择：**
  1. **可以使用 XML 创建部署描述符启动 Spring**
  2. **也可以在 `javax.servlet.ServletContainerInitializer` 中通过编程的方式启动。**

### JavaEE 环境下的启动

#### 部署描述符启动
传统的 Spring Framework 应用程序总是使用 Java EE 的部署描述符启动。配置文件中至少包含一个 `DispatcherServlet` 的实例，然后以 `contextConfugLocation` 初始化参数的形式为它提供配置文件。也可以包含多个 `DispatcherServlet` 实例。另外，一般还会配置 `ContextLoaderListener` 实例加上 `contextConfigLocation`的上下文参数。参数为其典型的配置如下：
```
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>WEB-INF/rootContext.xml</param-value>
</context-param>>
<listener>
    <listener-class>
org.springframework.web.context.ContextLoaderListener
    <listener-class>
</listener>
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.sringframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>contextConfigLocation</init-param>
    <param-value>WEB-INF/servletContext.xml</param-value>
<servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```
`ContextLoaderListener` 将在 WEB 应用程序启动时被初始化（因为它实现了 `ServletContextListener`)，然后从 `contextConfigLocation` 上下文初始化参数指定的配置文件中加载根应用上下文，并启动根应用上下文。

注意： `contextConfigLocation` 上下文初始化参数不同于 `DispatcherServlet` 的 `contextConfigLocation` Servlet初始化参数。它们不冲突；前者作用于整个 Servlet 上下文，而后者只作用于它所指定的 Servlet 。 监听器创建的根应用上下文将被自动设置为所有通过 `DispatcherServlet` 创建的应用上下文的父亲上下文。

#### 初始化器中使用编程的方式启动 Spring
`ServletContextListener` 可以以编程的方式配置应用程序的中 Servlet、 Listener 和 Filter 。使用该接口的缺点是：监听器的 `contextInitialized` 方法可能在其它监听器之后调用。 Java EE 6 中添加了一个新的接口 `ServletContainerInitializer` 。 实现了 `ServletContainerInitializer` 接口的类将在程序启动时，并在所有监听器启动之前调用它的 `onStartup` 方法。这是应用程序生命周期中最早可以使用的时间点。但是，不要再部署描述符中配置 `ServletContainerInitializer` ，相反，需要使用 Java 的服务提供接口（SPI：Service Provider Interface）声明实现了 `ServletContainerInitializer` 的一个或多个类，在文件 `/META-INF/services/javax.servlet.ServletContainerInitializer` 中列出它们，每行一个类。

这种方式不利的一面在于文件不能直接存在于应用程序的 WAR 文件或解压后的目录中——不能将文件放在 Web 应用程序的 `/META-INF/services` 目录中。它必须在 JAR 文件的 `/META-INF/services` 目录中，并且需要将该JAR文件包含在应用程序WAR的 `WEB-INF/lib` 目录中。     

Spring Framework 提供了一个桥接口，是这种方式更容易实现。 `org.springframework.web.SpringServletContainerInitializer` 实现了 `ServletContainerInitializer` 接口，而且包含该类的 JAR 中包含了一个服务提供文件，列出了 `SpringServletContainerInitializer` 类的名字，所以应用程序会在启动时调用它的 `onStartup` 方法。然后，该类将在此方法中扫描应用程序以寻找实现了 `org.springframework.web.WebApplicationInitializer` 接口的实现类，并调用所有找到类的 `onStartup` 方法。在 `WebApplicationInitializer` 实现类中可以配置 Servlet 、 Filter 和 Listener ，更重要的是，可以在该方法中启动 Spring 。

## Spring Framework 的配置方式
Spring Framework 的配置大致可以分为三种方式：

### 创建 XML 配置
这是最传统的配置方式，使用 &lt;beans&gt; XML 命名空间，将需要注入的 bean 配置在 ,&lt;bean&gt; 标签下即可配置 bean 。

### 创建混合配置
XML 配置文件的缺点是太繁杂，一个大型的企业级应用中，可能会定义数百个 bean ,每个 bean 都至少要三行代码的话，关是配置文件都要数千行。

spring 注解配置的核心在于 **组件扫描** 和 **注解配置**。 
#### 组件扫描的开启
XML 文件中使用 `<context:annotation-config>` 和 `<context:component-scan>` 元素即可开启**组件扫描** 和 **注解配置** 。不过`<context:component-scan>` 也有 `<context:annotation-config>` 注解的功能，因此只要配置 `<context:component-scan>` 就能开启上述功能

#### 扫描注解配置
通过使用注解扫描， Spring 将扫描通过特定注解指定的包去扫描类。扫描路径下的所有标注了 `@Componengt` 注解的类都将变成由 Spring 管理的 Bean，这意味着 Spring 将负责实例化它们并注入它们的依赖对象。

其它符合组件扫描的注解： 所有标注了 `@Component` 的注解都将变成组件扫描注解，任何标注了一个组件扫描注解的注解也将编程组件扫描注解。因此， 标注了 `@Controller` 、 `@Service`  、 `@Repository` 注解的类也将编程Spring 管理的bean。

与组件扫描注解配合使用的另一个注解是 `@Autowired` 。可以为任何公开、保护和私有的字段或接受一个或多个参数的 **设置方法** 标注 `@Autowired`。 `@Autowired` 声明了 Spring 实例化之后应该注入的依赖对象，可以指定当前bean定义采用某种类型的自动绑定模式。这样，你就无需手工明确指定该bean定义相关的依赖关系，从而也可以免去一些手工输入的工作量。并且它也可以用于构造器，通常Spring管理的bean类必须有无参构造函数，但对于只含有一个标注了 `@Autowired` 的构造器的类， Spring 将使用该构造器并注入所有的构造器参数。

Spring提供了5种自动绑定模式，即 `no、byName、byType、 constructor 和 autodetect`:
+ no ： 容器默认的自动绑定模式，也就是不采用任何形式的自动绑定，完全依赖手工明确配置各个bean之间的依赖关系。
+ byName: 按照类中声明的实例变量的名称，与XML配置文件或注解配置中声明的bean定义的beanName的值进行匹配，相匹配的bean定义将被自动绑定到当前实例变量上。
+ byType: 容器会根据当前bean定义类型，分析其相应的依赖对象类型，然后到容器所管理的所有bean定义中寻找与依赖对象类型相同的bean，将找到的符合条件的bean自动绑定到当前bean
+ constructor: 它与byType类型的绑定模式类似。不过，constructor是匹配构造方法的参数类型，而不是实例属性的类型。
+ autodetect： 这种模式是byType和constructor模式的结合体，如果对象拥有默认无参数的构造方法，容器会优先考虑byType的自动绑定模式。否则，会使用constructor模式


任何情况下，如果Spring无法为依赖找到匹配的bean，将抛出异常并启动失败。

同样，如果为依赖找到多个匹配的 bean，它也将抛出异常并启动失败。这种情况下，可以使用 `@Qualifier` 或 `@Primary` 注解解决。 通过 `@Qualifier` 可以使用指定名字的依赖。而使用 `@Primary` 标注的类表示在出现多个符合条件的依赖时，应该优先使用标注了该注解的bean。

### 使用 @Configuration 配置
上述两种配置方式基本上都还是要依赖 XML ，第二种方式中要在 XML 中开启注解组件扫描和启用注解配置功能。不过，在使用 XML 配置Spring时有一些缺点：
1. XML 难于调试。
2. 不能对 XML 配置进行单元测试。因为 XML 配置会启动整个应用程序并加载所有 bean ，实际上不是单元测试而是继承测试。

使用 `AnnotationConfigWebApplicationContext` 启动 Spring ，并调用该类的 `register(Class<T> configClass)` 方法注册配置类即可实现 java 配置。 这些配置类必须标注上 `@Configuration` 注解，也必须有默认构造函数，配置类中标注了 `@Bean` 注解的无参方法将注册 bean。

在配置类中可以使用下列注解代替 XML 中的一些配置元素：
1. `@ComponentScan` : 替代的是 `<context:component-scan>` ，启用组件扫描功能
2. `@EnableAspectAutoProxy` : 替代的是 `<aop:aspect-autoprooxy>` 元素，启用对标注了 `@Aspect` 注解的类的处理，面向切面编程时使用
3. `@EnableAsync` : 替代的是 `<async:*>` 命名空间，启用Spring的异步 `@Async` 方法执行
4. `@EnableCaching` : 替代的是 `<cache:*>` 命名空间。
5. `@EnableScheduling` : 替代的是 `<task:*>` 命名空间，激活标注了 `@Scheduled` 的计划的执行。
6. `@EnableTransactionManagement` : 替代的是 `<tx:annotation-driven>`， 可以为标注了 `@Transactional` 注解的方法启用事务管理。
7. `@EnableWebMvc` : 替代的是 `<mvc:annotation-driven>`， 激活了注解驱动的控制器请求映射。该注解将激活一个非常复杂的配置，通常需要进行自定义。通过使标注了 `@Configuration` 注解的配置里实现 `WebMvcConfigurer` 或者继承 `WebMvcConfigurerAdapter` 来定义整个 Web MVC 配置。

另外，某些情况下，我们不得不使用 XML 配置，比如要注入某个 Jar 包中的 Bean ，无法为该 bean 的类标注上类似 `@Component` 的注解。我们可以使用 `@Import` 和 `@ImportResource` 注解来加载其它配置类或 XML 配置文件。
```
//Java注解的方式配置 Bean
@Configuration
@Import({DatabaseConfiguration.class, ClusterConfiguraton.class})
@ImportResource("classpath:com/wrox/config/spring-security.xml)
@ComponentScan("com.baicy.wang")
public class ExampleConfiguration{
    @Bean
    public Bean getBean(){
        return new Bean();
    }
    .....
}
//使用上述配置启动上下文
AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
context.register(ExampleConfiguration.class);
```