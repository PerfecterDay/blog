#  Spring IOC 基于注解的配置
 {docsify-updated}
> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config.html

基于注解的注入在 XML 注入之前执行。因此，XML 配置会覆盖通过这两种方法注入的属性。

基于注解配置功能主要依赖 `BeanPostProcessor/BeanFactoryPostProcessor` 扩展来实现的，一些典型的 `BeanPostProcessor/BeanFactoryPostProcessor` 如下：
+ ConfigurationClassPostProcessor
+ AutowiredAnnotationBeanPostProcessor
+ CommonAnnotationBeanPostProcessor
+ PersistenceAnnotationBeanPostProcessor
+ EventListenerMethodProcessor

大致上有两类注解：
1. 一种是声明 bean 的注解，这种注解是用来配置 bean 的元数据，用于在容器中定义一个 bean
2. 另一种是在定义 bean 的依赖关系时（注入 bean）用到的注解，用于在一个 bean 中注入另一个 bean

## 声明 bean 相关的注解

### `@Component, @Repository, @Service, @Controller, @Configuration, @Named(JSR)`
这些注解都可以在 spring 容器中声明一个 bean ，前提是开启了 `@ComponentScan` 并且注解所在的包在注解扫描的包路径下。

以上的每个注解除了 `@Named` 都可以使用name 属性为 bean 命名，`@Named` 注解使用 value 属性为 bean 命名。
如果您不想依赖默认的 Bean 命名策略，您可以提供自定义的 Bean 命名策略。首先，实现 `BeanNameGenerator` 接口，并确保包含默认的无参数构造函数。然后，在配置扫描器时提供完全合格的类名，如下面的注解和 Bean 定义示例所示。

### `@ComponentScan`
只在类上加上上述注解并不能让Spring注册对应类型的Bean，只有开启了Spring 的扫描功能，Spring 才会扫描这些注解并且生成 bean 。  
为了自动检测这些类并注册相应的Bean，您需要在 `@Configuration` 类中添加 `@ComponentScan` ，其中 `basePackages` 属性是所有要扫描的类的包。(可以指定一个以逗号、分号或空格分隔的列表，其中包括每个要扫描的包）。

默认情况下，使用 `@Component、@Repository、@Service、@Controller、@Configuration` 或本身使用 `@Component` 注解的类都会被扫描到。然而，您可以通过应用自定义过滤器来修改和扩展该行为。添加它们作为`@ComponentScan` 注解的 `includeFilters` 或 `excludeFilters` 属性（或作为XML配置中<context:include-filter />或<context:exclude-filter />元素的<context:component-scan>子元素）。每个过滤元素都需要 `type` 和 `expression` 属性。
```
@Configuration
@ComponentScan(basePackages = "org.example",
		includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
		excludeFilters = @Filter(Repository.class))
public class AppConfig {
	// ...
}
```

### `@Bean`
`@Bean` 注解标识了生产bean的工厂方法和其他Bean定义属性，例如通过 `@Qualifier` 注解标识了限定符值。可以指定的其他方法级注解有 `@Scope` 、`@Lazy` 和自定义限定符注解。

普通Spring `@Component` 组件中的@Bean方法与Spring `@Configuration` 类中的@Bean方法处理方式不同。不同之处在于，`@Component` 类没有使用CGLIB来拦截方法和字段的调用。CGLIB 代理是调用 `@Configuration` 类中 @Bean 方法内的方法或字段时创建协作对象的 bean 元数据引用的一种方法。这些方法不是以正常的Java语义调用的，而是通过容器来提供Spring Bean的常规生命周期管理和代理，即使是通过对 `@Bean` 方法的编程调用来引用其他Bean时也是如此。与此相反，在普通的 `@Component` 类中调用@Bean方法中的方法或字段具有标准的Java语义，没有特殊的CGLIB处理或其他约束。

### `@Scope`
用来声明 bean 的作用范围， @Scope 注解仅能用在具体的 bean 类（对于有注解的组件）或工厂方法（对于 @Bean 方法）上。
<center><img src="pics/bean-scope.png" width="60%" ></center>


## 声明依赖关系的注解
+ `@Autowired`


<center><img src="pics/spring-ioc.png" width="60%" ></center>

### `@Inject (JSR 330)/@Autowired`
默认是根据bean类型注入的，并且默认依赖是必须的，如果找不到依赖可用，Spring会报错。如果依赖是非必需的，可以指定 `@Autowired(required = false)`，或者将依赖类型声明为 `Optional`，或者加上 `@Nullable` 注解。

可以用在构造方法上，为构造方法自动注入参数依赖。从Spring Framework 4.3开始，如果目标Bean一开始就只定义了一个构造函数，那么在这样的构造函数上就不再需要`@Autowired` 注解。然而，如果有几个构造函数，而且没有默认构造函数，那么至少有一个构造函数必须用 `@Autowired` 注解，以便指示容器使用哪一个。

`@Autowired` 注解应用于传统的setter方法。
你也可以将 `@Autowired` 注解应用于具有任意名称和多个参数的方法上。
你也可以将 `@Autowired` 应用于成员字段上。
当 `@Autowired` 用于集合类型时，Spring 会把所有集合指定的类型 bean 注入到集合中，当注入到 `Map` 时， key 是 beanName， value 是 bean。

注意：
`@Autowired、@Inject、@Value和@Resource`注解是由Spring `BeanPostProcessor` 实现处理的。这意味着你不能在你自己的 `BeanPostProcessor` 或 `BeanFactoryPostProcessor` 类型（如果有的话）中应用这些注解。这些注解默认只能用在普通 bean 声明中。


### `@Primary` 
当有多个同类型的 bean 声明时，必需指定一个作为主要依赖，否则 `@Autowired` 声明依赖时，会找到多个依赖无法决定注入哪个，然后就会报错。

### `@Qualifier/@Named (JSR 330)`
解决多个依赖注入问题时，除了给被依赖项的某项加上 `@Primary` 注解外，还可以使用 `@Qualifier/ @Named` 注解指定依赖的 bean 名字。有时候不能更改被依赖项的源码的时候，可以用这种方式。

甚至可以使用 `CustomAutowireConfigurer` 来实现自定义的 qualifier 逻辑：
```
<bean id="customAutowireConfigurer"
		class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
	<property name="customQualifierTypes">
		<set>
			<value>example.CustomQualifier</value>
		</set>
	</property>
</bean>
```

### `@Resource(JSR-250)`
可以用在字段属性或者 setter 方法上，默认是根据bean名字注入的，只有找不到特定名字的 bean 时，才会根据类型注入。
因此，在下面的示例中， tradeLoginService 字段首先查找名为 "tradeLoginService" 的 bean，如果发现 tradeLoginService 名字的 bean 不是 TradeLoginServiceV2 类型，就会报错。
然后返回到 TradeLoginServiceV2 类型的主类型匹配。
```
	//会注入名字为 tradeLoginService 的 bean，如果这个bean 不是 TradeLoginServiceV2 类型就会报错
    @Resource
    private TradeLoginServiceV2 tradeLoginService;

	//会注入名字为 tradeLoginServiceV2 的 bean
	@Resource(name = "tradeLoginServiceV2")
    private TradeLoginServiceV2 tradeLoginService2;

	@Resource
	private TradeLoginServiceV2 tradeLoginService3;

```

### `@PostConstruct` 和 `@PreDestroy`
`CommonAnnotationBeanPostProcessor` 不仅可以识别 `@Resource` 注解，还可以识别JSR-250生命周期注解：`jakarta.annotation.PostConstruct` 和 `jakarta.annotation.PreDestroy` 。在Spring 2.5中引入，对这些注解的支持为初始化回调和销毁回调中描述的生命周期回调机制提供了一个替代方案。只要在Spring应用上下文中注册了 `CommonAnnotationBeanPostProcessor` ，携带这些注解之一的方法就会在生命周期中与相应的Spring生命周期接口方法或明确声明的回调方法在同一时间被调用。

是 Spring `InitializingBean` 和 `DisposableBean` 接口的替代解决方案，也是JSR标准注解。

## 属性资源文件相关

两个重要的类： `PropertyPlaceholderConfigurer(标记为淘汰了)` 和 `PropertySourcesPlaceholderConfigurer` 。

### `@PropertySource`
`@PropertySource` 注解为在Spring的 `Environment` 中添加 `PropertySource` 提供了一个方便的声明机制。

在 `@PropertySource` 资源位置中出现的任何 `${...}` 占位符都将根据已在环境中注册的属性源集合进行解析。
给定一个名为 app.properties 的文件，其中包含键值对 testbean.name=myTestBean，下面的 `@Configuration` 类使用 `@PropertySource` 的方式使调用 testBean.getName() 返回 myTestBean：
```
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

 @Autowired
 Environment env;

 @Bean
 public TestBean testBean() {
  TestBean testBean = new TestBean();
  testBean.setName(env.getProperty("testbean.name"));
  return testBean;
 }
}
```

### `@Value`
使用 `@Value(${property.val:defaultVal})` 的形式注入属性值。 `${property.val:defaultVal}` 这种形式的属性引用不仅可以用在Java代码的Bean中注入属性，同样可以在属性文件中引用另一处属性值。