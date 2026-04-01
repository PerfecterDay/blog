# Classpath Scanning and Managed Components
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html

本章中的多数示例均采用XML来指定生成Spring容器内每个 `BeanDefinition` 的配置元数据。前一节（基于注解的容器配置）演示了如何通过源代码级注解提供大量配置元数据。然而即便在这些示例中，"基础"Bean定义仍需在XML文件中显式定义，注解仅驱动依赖注入机制。

本节描述了一种通过扫描类路径隐式检测候选组件的选项。候选组件是指符合过滤条件且在容器中注册了对应 Bean 定义的类。这消除了使用 XML 进行 Bean 注册的需求。取而代之的是，您可以使用注解（例如 `@Component` ）、 `AspectJ` 类型表达式或自定义过滤条件来选择哪些类在容器中注册了 Bean 定义。

可以使用Java而非XML文件来定义Bean: `@Configuration` 、 `@Bean` 、 `@Import` 和 `@DependsOn` 注解。

## @Component 和衍生注解
Spring 提供了更多类型的注解：`@Component` 、 `@Service` 和 `@Controller` 。 `@Component` 是适用于任何 Spring 管理组件的通用类型。 `@Repository` 、 `@Service` 和 `@Controller` 则是 `@Component` 的特化形式，分别在持久层、服务层和展示层。因此，虽然可以使用 `@Component` 标注组件类，但若改用 `@Repository` 、 `@Service` 或 `@Controller` 标注，这些类将更适于工具处理或与切面关联。例如，这些注解是切入点的理想目标。在 Spring 框架的未来版本中， `@Repository` 、 `@Service`  和 `@Controller` 可能承载更多语义。因此，若需在服务层选择 `@Component` 或 `@Service` ， `@Service` 显然是更优选项。同理， `@Repository` 注解用于标记任何实现存储库（也称为数据访问对象或DAO）角色或模式的类。该标记注解的功能之一是自动转换异常。

## 作为元注解组合使用
Spring 提供的许多注解都可以在开发者的代码中作为元注解使用。元注解是指可应用于其他注解的注解。例如，前文提到的 `@Service` 注解就带有 `@Component` 元注解，如下例所示：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

	// ...
}
```

还可以组合元注解来创建“组合注解”。例如，Spring MVC中的 `@RestController` 注解由 `@Controller` 和 `@ResponseBody` 组合而成。

此外，组合注解可选择性地重新声明元注解的属性以实现自定义。当仅需暴露元注解属性的子集时，此特性尤为实用。例如，Spring的 `@SessionScope` 注解将作用域名称硬编码为 `session` ，但仍允许自定义 `proxyMode` 属性。以下代码片段展示了 `@SessionScope` 注解的定义：
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

	/**
	 * Alias for {@link Scope#proxyMode}.
	 * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
	 */
	@AliasFor(annotation = Scope.class)
	ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

在使用的时候可以覆盖默认属性：
```
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
	// ...
}
```

## 自动扫描与注册
Spring能够自动检测到特定注解标注的类，并将对应的 `BeanDefinition` 实例注册到 `ApplicationContext` 中。  
要自动检测一些类并注册对应的Bean，需要在 `@Configuration` 类中添加 `@ComponentScan` 注解，其中 `basePackages` 属性需配置为想要注册类所在的包或父包，可以指定以**逗号、分号或空格分隔的列表**，该列表包含每个类的父包。
```
@Service
public class SimpleMovieLister {

	private final MovieFinder movieFinder;

	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}

@Configuration
@ComponentScan(basePackages = "org.example,org.demo")
public class AppConfig  {
	// ...
}
```

### 属性占位符与Ant-style模式
`@ComponentScan` 中的 `basePackages` 和 `value` 属性支持 `${…}` 属性占位符，这些占位符将根据 `Environment` 中的属性值进行解析，同时支持 Ant 风格:  `org.example.**`。  

此外，可以单独或在单个字符串中指定多个包或模式——例如，`{"org.example.config", "org.example.service.**"}` 或 `"org.example.config, org.example.service.**"` 。
```
@Configuration
@ComponentScan("${app.scan.packages}")
public class AppConfig {
	// ...
}

app.scan.packages=org.example.config, org.example.service.**
```

### 指定过滤器
默认情况下，仅检测标注有 `@Component` 、 `@Repository` 、 `@Service` 、 `@Controller` 、 `@Configuration` 或自带 `@Component` 注解的自定义注解的类作为候选组件。但可通过应用自定义过滤器来修改和扩展此行为。在 `@ComponentScan` 注解中添加 `includeFilters` 或 `excludeFilters` 属性（或在XML配置中作为 `<context:component-scan>` 元素的 `<context:include-filter />` 或 `<context:exclude-filter />` 子元素）。每个过滤器元素都需要 `type` 和 `expression` 属性。下表描述了过滤选项：
**Table 1. Filter Types**

| Filter Type | Example Expression | Description |
|------------|--------------------|-------------|
| annotation (default) | `org.example.SomeAnnotation` | An annotation to be present or *meta-present* at the type level in target components. |
| assignable | `org.example.SomeClass` | A class (or interface) that the target components are assignable to (extend or implement). |
| aspectj | `org.example..*Service+` | An AspectJ type expression to be matched by the target components. |
| regex | `org\.example\.Default.*` | A regex expression to be matched by the target components' class names. |
| custom | `org.example.MyTypeFilter` | A custom implementation of the `org.springframework.core.type.TypeFilter` interface. |

比如：
```
@Configuration
@ComponentScan(basePackages = "org.example",
		includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
		excludeFilters = @Filter(Repository.class))
public class AppConfig {
	// ...
}
```

此外， `@ComponentScan` 注解还有以下属性在一些场景下可以使用：
+ `useDefaultFilters` ： 默认 `true` , 设置为 `false` 时， `@Component` `@Repository` , `@Service` , `@Controller` 这些注解将不会被扫描注册
+ `lazyInit` : 默认 `false` ，设置为 `true` 时，将会启动延迟初始化

### bean 命名
当组件在扫描过程中被自动检测时，其Bean名称由该扫描器配置的 `BeanNameGenerator` 策略生成。默认情况下，系统会使用 `AnnotationBeanNameGenerator` 。对于 Spring 类型的注解，若通过注解的 `value` 属性提供名称，该名称将被用作对应 bean 的名称。当使用 `@jakarta.inject.Named` 注解替代 Spring 类型注解时，此约定同样适用。

从 Spring Framework 6.1 开始，用于指定 Bean 名称的注解属性名称不再强制要求为 `value` 。自定义注解可以声明名称不同的属性（例如 name），并使用 `@AliasFor(annotation = Component.class, attribute = "value")` 对其进行标注。具体示例参见 `ControllerAdvice#name()` 的源代码声明。

若不想依赖默认的 Bean 命名策略，可提供自定义命名策略。首先实现 `BeanNameGenerator` 接口，并确保包含默认的无参构造函数。随后在配置扫描器时提供全限定类名，如下例注解和 Bean 定义所示：
```
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
	// ...
}
```

通常情况下，当其他组件可能显式引用该名称时，建议添加注解时显式指定名称。另一方面，当容器负责连接时，自动生成的名称已足够使用。


### 为自动扫描的组建添加 Scope
与Spring管理的组件类似，自动检测组件的默认且最常见作用域为单例（ `singleton` ）。但有时需要指定其他作用域，可通过 `@Scope` 注解实现。如下例所示，可在注解内提供作用域名称：
```
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
```

`@Scope` 注解仅在具体 Bean 类（针对注解组件）或工厂方法（针对 `@Bean` 方法）上有效。与 XML Bean 定义不同，这里不存在 Bean 定义继承的概念，类层面的继承层次结构对元数据而言无关紧要。  

与这些作用域的预构建注解类似，也可以通过Spring的元注解方法组合自己的作用域注解：例如，使用 `@Scope("prototype")` 元注解的自定义注解，声明自定义的作用域为 `prototype` 。

要提供自定义的作用域解析策略而非依赖基于注解的方法，可实现 `ScopeMetadataResolver` 接口。请确保包含默认的无参构造函数。随后在配置扫描器时即可提供全限定类名，如下例所示（同时展示了注解和Bean定义两种方式）：
```
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
	// ...
}
```

在使用某些非单例作用域时，可能需要为作用域对象生成代理。为此， `component-scan` 元素提供了 `scoped-proxy` 属性，其取值有三种：`no` 、 `interfaces` 和 `targetClass` 。例如以下配置将生成标准JDK动态代理：
```
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
	// ...
}
```

### 指定 Qualifier
当依赖类路径扫描进行组件自动检测时，可通过候选类的类型级注解 `@Qualifier` 提供限定符元数据。以下三个示例演示了该技术：
```
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
```

## 在 Components 中定义 bean
Spring 扫描的组件中也可向容器提供bean定义元数据。 使用与在 `@Configuration` 注解类中定义bean元数据相同的 `@Bean` 注解实现此功能。以下示例展示了具体操作方式：
```
@Component
public class FactoryMethodComponent {

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	public void doWork() {
		// Component method implementation omitted
	}
}
```

也支持自 auto wiring 字段和方法，并额外支持自动连接 `@Bean` 方法。以下示例展示了具体实现方式：
```
@Component
public class FactoryMethodComponent {

	private static int i;

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	// use of a custom qualifier and autowiring of method parameters
	@Bean
	protected TestBean protectedInstance(
			@Qualifier("public") TestBean spouse,
			@Value("#{privateInstance.age}") String country) {
		TestBean tb = new TestBean("protectedInstance", 1);
		tb.setSpouse(spouse);
		tb.setCountry(country);
		return tb;
	}

	@Bean
	private TestBean privateInstance() {
		return new TestBean("privateInstance", i++);
	}

	@Bean
	@RequestScope
	public TestBean requestScopedInstance() {
		return new TestBean("requestScopedInstance", 3);
	}
}
```

从 Spring Framework 4.3 开始，还可以声明类型为 `InjectionPoint` （或其更具体的子类 `DependencyDescriptor` ）的工厂方法参数，以访问触发当前 Bean 创建的请求注入点。请注意，这仅适用于 Bean 实例的实际创建，而非现有实例的注入。因此，此功能对原型作用域的 Bean 最具意义。对于其他作用域，工厂方法仅能获取触发当前作用域内新 Bean 实例创建的注入点（例如触发懒加载单例 Bean 创建的依赖项）。在此类场景中，需谨慎处理注入点元数据的语义含义。以下示例展示了如何使用 `InjectionPoint` ：
```
@Component
public class FactoryMethodComponent {

	@Bean @Scope("prototype")
	public TestBean prototypeInstance(InjectionPoint injectionPoint) {
		return new TestBean("prototypeInstance for " + injectionPoint.getMember());
	}
}
```

### 与 @Configuration 的区别
常规Spring组件中的 `@Bean` 方法与Spring `@Configuration` 类内部的同类方法处理方式不同。区别在于 `@Component` 类不会通过CGLIB增强来拦截方法和字段的调用。CGLIB代理机制是 `@Configuration` 类中 `@Bean` 方法调用方法或字段时创建协作对象bean元数据引用的实现方式。此类方法的调用不遵循常规Java语义，而是通过容器实现Spring Bean的生命周期管理与代理机制——即使通过编程方式调用 `@Bean` 方法引用其他Bean时亦如此。反之，在普通 `@Component` 类中调用 `@Bean` 方法内的方法或字段时，则遵循标准Java语义，不涉及特殊CGLIB处理或其他限制。

请看以下示例：
```
@Component
public class ComponentTest {

    @Bean
    public A a(){
        System.out.println("aaa in componentTest");
        return new A();
    }

    @Bean
    public A b(){
        System.out.println("xxx in componentTest");
        return a();
    }
}

@Configuration
public class ConfigurationTest {
    @Bean
    public A x(){
        System.out.println("aaa in ConfigurationTest");
        return new A();
    }

    @Bean
    public A y(){
        System.out.println("xxx in ConfigurationTest");
        return x();
    }
}

@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public @Nullable Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return BeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
    }

    @Override
    public @Nullable Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization： "+beanName);
        return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
    }
}
```

输出结果：
```
aaa in componentTest
postProcessAfterInitialization： a

xxx in componentTest
aaa in componentTest
postProcessAfterInitialization： b

aaa in ConfigurationTest
postProcessAfterInitialization： x

xxx in ConfigurationTest
postProcessAfterInitialization： y
```

从结果可以看出，在 `@Component` 中一个 `@Bean` 方法调用另一个 `@Bean` 方法时，是 java 层面的直接调用，但是在 `@Configuration` 中，并没有调用另一个 `@Bean` 方法，因为已经有 bean 存在了，所以不会再次调用。


可以将@Bean方法声明为静态方法，这样无需实例化其所属的配置类即可调用这些方法。这种做法在定义后处理器Bean（例如 `BeanFactoryPostProcessor` 或 `BeanPostProcessor` 类型）时尤为合理，因为此类Bean会在容器生命周期早期初始化，此时应避免触发配置的其他部分。

由于技术限制（如本节前文所述），容器永远不会拦截对静态 `@Bean` 方法的调用——即使在 `@Configuration` 类中也是如此：**CGLIB子类化仅能覆盖非静态方法**。因此，直接调用其他 `@Bean` 方法将遵循标准Java语义，导致工厂方法直接返回独立实例。

`@Bean` 方法的Java语言可见性不会直接影响Spring容器中生成的bean定义。可在非 `@Configuration` 类中自由声明工厂方法，也可在任意位置声明静态方法。但需注意： `@Configuration` 类中的常规 `@Bean` 方法必须可被覆盖——即不得声明为 `private` 或 `final` 。

`@Bean` 方法还会在组件或配置类的基类中被发现，同时也会在组件或配置类实现的接口中声明的Java默认方法中被发现。这使得在构建复杂配置方案时具有极大灵活性，甚至可通过Java默认方法实现多重继承。

最后，单个类可包含多个针对同一 Bean 的 `@Bean` 方法，作为运行时根据可用依赖项选择使用的工厂方法。这与其他配置场景中选择"最贪婪"构造函数或工厂方法的算法相同：**构造时会选择满足最多依赖项的变体，类似于容器在多个  `@Autowired` 构造函数间进行选择的方式。**