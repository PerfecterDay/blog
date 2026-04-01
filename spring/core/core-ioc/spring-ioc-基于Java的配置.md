#  Spring IOC 基于Java的配置
 {docsify-updated}
> https://docs.spring.io/spring-framework/reference/core/beans/java.html

Spring Java 配置支持的核心工具是 `@Configuration` 注解和 `@Bean` 注解修饰的方法。

`@Bean` 注解用于表示一个方法会实例化、配置和初始化了一个由 Spring IoC 容器管理的Bean对象。可以在任何 Spring `@Component` 中使用 `@Bean` 注解方法。不过，它们最常用于 `@Configuration` 注解的 Bean 中。

用 `@Configuration` 来注解一个类，表明它的主要目的是作为 bean 定义的配置类。此外，`@Configuration` 类还可以通过调用同一类中的其他 `@Bean` 方法来定义 bean 之间的依赖关系。

当 `@Bean` 方法在未注明 `@Configuration` 的类中声明时，它们被称为以 "精简 "模式处理。在未注明 `@Configuration` 的 Bean 上声明的 `@Bean` 方法被认为是 "精简 "的，类的主要目的不是配置 Bean，而 `@Bean` 方法只是其中的一种附加功能。

与完整的 `@Configuration` 不同，精简版 `@Bean` 方法不能声明 bean 之间的依赖关系。取而代之的是，它们对其包含组件的内部状态进行操作，也可选择对其声明的参数进行操作。因此，这种 @Bean 方法不应调用其他 @Bean 方法。每个此类方法实际上只是特定 Bean 引用的工厂方法，没有任何特殊的运行时语义。这样做的积极副作用是，在运行时无需应用 CGLIB 子类化，因此在类设计方面没有任何限制（即包含的类可以是 final 类等等）。

## AnnotationConfigApplicationContext
Spring 3.0 中引入了 Spring `AnnotationConfigApplicationContext` 。这种多用途 `ApplicationContext` 实现不仅能接受 `@Configuration` 类作为输入，还能接受普通 `@Component` 类和使用 JSR-330 元数据注释的类。

当 `@Configuration` 类作为输入提供时，`@Configuration` 类本身会注册为 bean 定义，类中所有已声明的` @Bean` 方法也会注册为 bean 定义。
```
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
	ctx.register(AppConfig.class, OtherConfig.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```

### 启用组件扫描
要启用组件扫描，可以对 `@Configuration` 类进行如下注解：
```
@Configuration
@ComponentScan(basePackages = "com.acme") 
public class AppConfig  {
	// ...
}
```
或者也可以使用 `AnnotationConfigApplicationContext` 的 `scan(String packageStr)` 方法：
```
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.scan("com.acme");
	ctx.refresh();
	MyService myService = ctx.getBean(MyService.class);
}
```

### AnnotationConfigWebApplicationContext
`AnnotationConfigApplicationContext` 的 `WebApplicationContext` 变体通过 `AnnotationConfigWebApplicationContext` 提供。在配置 Spring `ContextLoaderListener` 服务程序监听器、Spring MVC `DispatcherServlet` 等时，可以使用该实现。
```
<web-app>
	<!-- Configure ContextLoaderListener to use AnnotationConfigWebApplicationContext
		instead of the default XmlWebApplicationContext -->
	<context-param>
		<param-name>contextClass</param-name>
		<param-value>
			org.springframework.web.context.support.AnnotationConfigWebApplicationContext
		</param-value>
	</context-param>

	<!-- Configuration locations must consist of one or more comma- or space-delimited
		fully-qualified @Configuration classes. Fully-qualified packages may also be
		specified for component-scanning -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>com.acme.AppConfig</param-value>
	</context-param>

	<!-- Bootstrap the root application context as usual using ContextLoaderListener -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- Declare a Spring MVC DispatcherServlet as usual -->
	<servlet>
		<servlet-name>dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<!-- Configure DispatcherServlet to use AnnotationConfigWebApplicationContext
			instead of the default XmlWebApplicationContext -->
		<init-param>
			<param-name>contextClass</param-name>
			<param-value>
				org.springframework.web.context.support.AnnotationConfigWebApplicationContext
			</param-value>
		</init-param>
		<!-- Again, config locations must consist of one or more comma- or space-delimited
			and fully-qualified @Configuration classes -->
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>com.acme.web.MvcConfig</param-value>
		</init-param>
	</servlet>

	<!-- map all requests for /app/* to the dispatcher servlet -->
	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/app/*</url-pattern>
	</servlet-mapping>
</web-app>
```

## @Bean注解
`@Bean` 是一个方法级注解，也可以用于其它注解上组合成自定义注解，是 XML <bean/> 元素的直接类似物。
```
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
	@AliasFor("name")
	String[] value() default {};
	@AliasFor("value")
	String[] name() default {};
	Autowire autowire() default Autowire.NO;
	boolean autowireCandidate() default true;
	String initMethod() default "";
	String destroyMethod() default AbstractBeanDefinition.INFER_METHOD;
}
```

该注解支持 <bean/> 提供的部分属性，例如:
+ `init-method`
+ `destroy-method`
+ `autowiring`
+ `name`

### 声明一个 Bean
要声明 bean，可以使用 `@Bean` 注解来注解方法。会将方法返回类型的对象注册到容器中。默认情况下，**Bean 名称与方法名称相同**，也可以显示指定 bean 名字，还可以指定多个名字。下面的示例显示了 `@Bean` 方法声明：
```
@Configuration
public class AppConfig {
	@Bean("tranService")
	public TransferServiceImpl transferService() {
		return new TransferServiceImpl();
	}

	@Bean({"tranService2","defaultTranService"})
	public TransferServiceImpl transferServiceTwo() {
		return new TransferServiceImpl();
	}
}
```

### 声明依赖
一个 `@Bean` 注解的方法可以有任意数量的参数来描述构建该 Bean 所需的依赖关系。例如，如果我们的 `TransferService` 需要 `AccountRepository` ，我们可以用一个方法参数来具体化这种依赖关系，如下例所示：
```
@Configuration
public class AppConfig {
	@Bean
	public TransferService transferService(AccountRepository accountRepository) {
		return new TransferServiceImpl(accountRepository);
	}
}
```
注入机制类似于构造函数依赖注入一致。另外，也可以使用 `initMethod` 、 `destroyMethod` 管理生命周期回调：
```
public class BeanOne {
	public void init() {
		// initialization logic
	}
}

public class BeanTwo {
	public void cleanup() {
		// destruction logic
	}
}

@Configuration
public class AppConfig {
	@Bean(initMethod = "init")
	public BeanOne beanOne() {
		return new BeanOne();
	}

	@Bean(destroyMethod = "cleanup")
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
```
直接使用 Java 配置时，我们可以随心所欲地控制对象，所以不必总是依赖容器生命周期相关机制，可以直接通过代码管理 Bean 的初始化逻辑。

## @Scope 注解
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Scope {
	@AliasFor("scopeName")
	String value() default "";

	@AliasFor("value")
	String scopeName() default "";

	ScopedProxyMode proxyMode() default ScopedProxyMode.DEFAULT;
}
```

### scoped-proxy
Spring IoC 容器不仅管理对象（Bean）的实例化，还管理协作者（或依赖关系）的连接。如果您想将（例如）HTTP 请求作用域的 Bean 注入到另一个作用域更长的 Bean 中，可以选择注入一个 AOP 代理来代替作用域 Bean。也就是说，需要注入一个代理对象，它与作用域对象公开相同的公共接口，但也能从相关作用域（如 HTTP 请求）检索真正的目标对象，并将方法调用委托给真正的对象。

当针对作用域原型的 Bean 声明 `<aop:scoped-proxy/>` 时，对共享代理Bean的每个方法调用都会导致创建一个新的目标实例（比如HttpRequest/Session等），然后将调用转发到该目标实例上。
```
@Bean
@SessionScope
public UserPreferences userPreferences() {
	return new UserPreferences();
}

@Bean(destroyMethod = "close",initMethod = "init")
@Scope(proxyMode=ScopedProxyMode.TARGET_CLASS,value = "prototype")
public A a(){
	System.out.println("-----AA---------");
	return new A();
}
```

## @Configuration 注解
```
@Configuration
public class AppConfig {

	@Bean
	public ClientService clientService1() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao());
		return clientService;
	}

	@Bean
	public ClientService clientService2() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao());
		return clientService;
	}

	@Bean
	public ClientDao clientDao() {
		return new ClientDaoImpl();
	}
}
```
在 `clientService1()` 和 `clientService2()` 中各调用了一次 `clientDao()` 。由于该方法会创建一个新的 `ClientDaoImpl` 实例并返回，因此通常会有两个实例（每个服务一个）。这肯定会有问题：在 Spring 中，实例化的 Bean 默认具有单例作用域。这就是神奇之处：**所有 `@Configuration` 类在启动时都会使用 CGLIB 进行子类强化。在子类中，子方法在调用父方法并创建新实例之前，会首先检查容器中是否有任何缓存的（作用域）Bean**。因此上例中实际上只会有一个 `ClientDao` 实例。 根据 bean 的作用域，行为可能会有所不同。这里讨论的是单子。

由于 CGLIB 会在启动时动态添加功能，因此有一些限制。特别是，配置类不能是最终类。不过，允许在配置类上使用任何构造函数，包括使用 @Autowired 或单个非默认构造函数声明进行默认注入。

如果您希望避免任何 CGLIB 施加的限制，请考虑在非 @Configuration 类（例如，在纯 @Component 类）上声明您的 @Bean 方法，或使用 @Configuration(proxyBeanMethods = false) 注释您的配置类。这样，@Bean 方法之间的跨方法调用就不会被拦截，因此您必须完全依赖构造函数或方法级别的依赖注入。

## @Import注解
`@Import` 注解允许从另一个配置类中加载 @Bean 定义
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
	/**
	 * {@link Configuration @Configuration}, {@link ImportSelector},
	 * {@link ImportBeanDefinitionRegistrar}, or regular component classes to import.
	 */
	Class<?>[] value();
}
```
假如项目中依赖了一个第三方包，这个包中有些 `@Configuration` 配置类没有在我们的扫描路径上，而我们又需要使用这些配置，那么可以在我们自己的配置类上使用 `@Import` 注解导入第三方的配置类。

当使用 `@Import(ConfigurationA.class)` 导入一个类 `ConfigurationA`　时，会使 `ConfigurationA` 成为一个 Bean 。相当于 `@Component` 注解修饰了 `ConfigurationA` 。任何声明一个 Bean 的行为同样适用于 `ConfigurationA` 。比如会调用 `xxxAware` 接口的方法、 `@PostConstruct` 修饰的方法也会在合适的时候调用、`@Bean` 修饰的方法会注入 Bean 到容器。
