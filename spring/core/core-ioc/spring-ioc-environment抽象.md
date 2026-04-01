# Spring Ioc Environment抽象
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/environment.html

`Environment` 接口是集成在容器中的一个抽象概念，它对应用环境的两个关键方面进行建模：`profiles` 和 `properties`。

`Profile` 是一组已命名的、符合逻辑的 Bean 定义，只有在给定的 `Profile` 处于 active 状态时才会在容器中注册。无论 bean 是用 XML `还是注解定义的，都可以分配给某个Profile` 。 `Environment` 对象保存了哪些 `Profile` 处于 active 状态，以及哪些 `Profile` 默认应处于活动状态。

`Properties` 在几乎所有应用程序中都扮演着重要角色，其来源可能多种多样：属性文件、JVM 系统属性、系统环境变量、JNDI、Servlet 上下文参数、 `Properties` 对象、 `Map` 对象等。 `Environment` 对象在属性方面的作用是为用户提供一个方便的服务接口，用于配置属性源并从中解析属性。

## Profile
`Profile` 在核心容器中提供了一种机制，允许在不同的 `Profile` 中注册不同的 Bean。比如：我们想在开发环境使用内存数据库，但是在生产/QA环境使用 JNDI 数据库。
```
@Configuration
public class AppConfig {

	@Bean("dataSource")
	@Profile("development") 
	public DataSource standaloneDataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.addScript("classpath:com/bank/config/sql/test-data.sql")
			.build();
	}

	@Bean("dataSource")
	@Profile("production") 
	public DataSource jndiDataSource() throws Exception {
		Context ctx = new InitialContext();
		return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
	}
}
```

### 注解-@Profile
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {
	String[] value();

}
```
profile字符串可包含一个简单的profile名称（例如， `production` ）或一个profile表达式。profile 表达式允许表达更复杂的 profile 逻辑。profile表达式支持以下操作符：
+ !: 逻辑非 NOT
+ &: 多个 profile 间进行逻辑与 AND
+ |: 多个 profile 间进行逻辑或 OR

```
@Profile({"A","B","C | D"})
@Profile("production & us-east")
```

还可以将 `@Profile` 作为元注解来定义自定义注解：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

如果某个 `@Configuration` 类被标记为 `@Profile` ，则该类关联的所有 `@Bean` 方法和 `@Import` 注解都会被跳过，除非指定的 profile 中有一个或多个处于活动状态。如果某个 `@Component` 或 `@Configuration` 类被标记为 `@Profile({"p1", "p2"})` ，则该类不会被注册或处理，除非 profile  'p1' 或 'p2' 已被激活。若 profile 前缀为否定运算符 (`!`)，则仅当该 profile 未激活时才会注册注解元素。例如，对于 `@Profile({"p1", "!p2"})` ，当 profile  'p1' 激活或 'p2' 未激活时将触发注册。

`@Profile` 注解也可在方法级别声明，用于仅包含某个配置类中的特定 Bean（例如用于特定 Bean 的替代变体）。

在 `@Bean` 方法上使用 `@Profile` 时，可能存在特殊情况：当存在同名重载的 `@Bean` 方法（类似于构造函数重载）时，所有重载方法都必须一致声明 `@Profile` 条件。若条件不一致，则仅重载方法中首次声明的条件生效。因此， `@Profile` 无法用于根据特定参数签名在重载方法间进行选择。同一Bean的所有工厂方法在创建时的解析遵循Spring的构造函数解析算法。

若需定义具有不同 profile 的替代Bean，请使用不同的Java方法名，通过 `@Bean` 的 `name` 属性指向同一Bean名称，如前例所示。若所有参数签名完全一致（例如所有变体均采用无参数工厂方法），这本就是唯一能在有效Java类中实现此类配置的方式（因为特定名称和参数签名的方法只能存在一个）。

### 激活 Profile
激活 Profile 有多种方法，但最直接的方法是通过 `ApplicationContext` 使用 `Environment` 的 API 以编程方式进行激活。下面的示例展示了如何进行激活：
```
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

此外，还可以通过 `spring.profiles.active` 属性声明 Profile ，该属性可通过系统环境变量、JVM 系统属性、web.xml 中的 servlet 上下文参数或 JNDI 中的条目指定。在集成测试中，可通过使用 `spring-test` 模块中的 `@ActiveProfiles` 注解来声明活动 profile 。可以同时激活多个 profile 。
```
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
-Dspring.profiles.active="profile1,profile2"
```

### Default Profile
在没有激活任何 profile 的情况下会启用 defaule profile。
```
@Configuration
@Profile("default")
public class DefaultDataConfig {
	@Bean
	public DataSource dataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.build();
	}
}
```
如果没有指定其它 active profile，则会创建该数据源。如果激活了任何其它 profile ，则 default profile 不会激活。

默认 profile 的名称是 default，可以通过在 `Environment` 上使用 `setDefaultProfiles()` 或声明使用 `spring.profiles.default` 属性来更改默认 profile 的名称。

## Properties

### PropertySource 抽象
Spring 的 `Environment` 提供了对可配置的 `PropertySource` 层次结构的搜索操作。
```
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```
在前面的代码段中，向 Spring 询问当前 `Environment` 是否定义了 `my-property` 属性的高级方法。为了回答这个问题，环境对象会对一组 `PropertySource` 对象进行搜索。 `PropertySource` 是对任何 `key-value` 来源的简单抽象。

Spring 的 `StandardEnvironment` 配置了两个 `PropertySource` 对象，一个代表 JVM 系统属性集（`System.getProperties()`），另一个代表系统环境变量集（`System.getenv()`）。

用于web环境的 `StandardServletEnvironment` 中还增加了其他默认属性源，包括 servlet 配置、servlet 上下文参数，以及 JNDI 可用时的 `JndiPropertySource` 。

`Environment` 搜索是分层进行的。默认情况下，系统属性优先于环境变量。因此，如果在调用 `env.getProperty("my-property")` 时， `my-property` 属性恰好在两个地方都被设置了，那么系统属性值将 "胜出" 并返回。请注意，属性值不会被合并，而是会被优先级高的条目完全覆盖。

对于 `StandardServletEnvironment` ，优先级递减的顺序如下：
1. ServletConfig 参数
2. ServletContext 参数
3. JNDI 环境变量（java:comp/env/ 条目）
4. JVM 系统属性（-D 命令行参数）
5. JVM 系统环境（操作系统环境变量）

最重要的是，整个机制是可配置的。可以自定义自己的 `PropertySource` ，并将其添加到当前环境的 `PropertySources` 中：
```
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```
在上面的代码中， `MyPropertySource` 在搜索中的优先级最高。如果 `MyPropertySource` 包含 `my-property` 属性，则会检测并返回该属性，而不会返回其他 `PropertySource` 中的 `my-property` 属性。 `MutablePropertySources` API 提供了许多方法，允许对属性源集合进行精确操作。

### 注解 @PropertySource
`@PropertySource` 注解为在 Spring 的环境中添加 `PropertySource` 提供了一种方便的声明式机制。

给定一个名为 app.properties 的文件，其中包含键值对 `testbean.name=myTestBean` ，下面的 `@Configuration` 类使用 `@PropertySource` 的方式是调用 `testBean.getName()` 返回 myTestBean：
```
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
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
在 `@PropertySource` 资源位置中出现的任何 `${...}` 占位符都会根据已在环境中注册的属性源集合进行解析，如下例所示：
```
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

 @Autowired
 Environment env;
}
```
假设 `my.placeholder` 存在于已注册的某个属性源中，该占位符将被解析为相应的值。如果没有，则使用 `default/path` 作为默认值。如果未指定默认值且无法解析属性，则会抛出 `IllegalArgumentException` 异常。



ConfigurationClassPostProcessor

### 各种配置及其优先级
```
┌──────────────────────────────────────────────┐
│ Command Line Arguments                       │
│                                              │
│ java -jar app.jar                            │
│   --server.port=8081                         │
│   --spring.profiles.active=uat               │
└──────────────▲───────────────────────────────┘
               │  最高优先级
               │
┌──────────────┴───────────────────────────────┐
│ JVM System Properties                         │
│                                              │
│ java -Dserver.port=8082                      │
│      -Dspring.profiles.active=uat            │
│                                              │
│ System.getProperty("server.port")            │
└──────────────▲───────────────────────────────┘
               │
┌──────────────┴───────────────────────────────┐
│ OS Environment Variables                     │
│                                              │
│ export SERVER_PORT=8083                      │
│ export SPRING_PROFILES_ACTIVE=uat            │
│                                              │
│ System.getenv("SERVER_PORT")                 │
└──────────────▲───────────────────────────────┘
               │
┌──────────────┴───────────────────────────────┐
│ application.yml / application.properties     │
│                                              │
│ application-uat.yml                          │
│   server.port: 8084                          │
└──────────────▲───────────────────────────────┘
               │
┌──────────────┴───────────────────────────────┐
│ @PropertySource / 默认值                     │
│                                              │
│ @Value("${server.port:8080}")                │
└──────────────▲───────────────────────────────┘
               │  最低优先级
               ▼
```