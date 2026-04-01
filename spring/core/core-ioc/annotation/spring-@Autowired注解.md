# @Autowired 注解
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html

```
注入机制（只有两种）
├── 构造函数注入
└── 属性注入
     ├── setter 注入
     └── 字段注入

@Autowired / @Inject / @Resource
└── 注入点声明（不是机制）
```

`@Autowired` 、 `@Inject` 、 `@Value` 和 `@Resource` 注解由 Spring `BeanPostProcessor`->`AutowiredAnnotationBeanPostProcessor` 的实现处理。所以不能在自己的 `BeanPostProcessor` 或 `BeanFactoryPostProcessor` 类型的 bean 中应用这些注解。

## 注解构造器
```
public class MovieRecommender {

	private final CustomerPreferenceDao customerPreferenceDao;

	@Autowired
	public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
		this.customerPreferenceDao = customerPreferenceDao;
	}

	// ...
}
```

任何给定 Bean 类的构造函数中：
+ 仅允许一个构造函数声明 `@Autowired` 注解且 `required` 属性设置为 true，这表明该构造函数在作为 Spring Bean 使用时将被自动装配。因此，如果 `required` 属性保留其默认值 `true` ，则仅允许单个构造函数被 `@Autowired` 注解标记。
  ```
    @Component
    public class A {
        public A() {
            System.out.println("default");
        }

        @Autowired
        public A(String a) {
            System.out.println("with string");
        }
    }
  ```
+ 若多个构造器声明该注解，则必须全部声明 `required=false` 才能成为自动装配候选对象（类似于 XML 中的 `autowire=constructor` ）。系统将选择依赖项最多且能被 Spring 容器匹配的 Bean 满足的构造器。若所有候选构造器均无法满足依赖，则使用 `primary` 构造器/默认构造器（若存在）。同理，
  ```
  @Component
  public class A {
      @Autowired(required = false)
      public A() {
          System.out.println("default");
      }
  
      @Autowired(required = false)
      public A(String a) {
          System.out.println("with string");
      }
  }
  ```
  注意，必须全部声明 `@Autowired(required = false)` ，否则会报 `BeanCreationException`

+ 若类声明多个构造器但均未标注 `@Autowired` ，则使用 `primary` 构造器/默认构造器（若存在）。
  ```
  @Component
  public class A {
    //使用默认构造器
      public A() {
          System.out.println("default");
      }
  
      public A(String a) {
          System.out.println("with string");
      }
  }
  ```
+ 若类仅声明单个构造器，则始终使用该构造器（即使未标注）。需注意：被注解的构造器不必声明为 `public` 。

## 注解 setter 方法
```
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Autowired
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```


## 注解任意的方法
```
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

## 注解字段属性
```
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    private MovieCatalog movieCatalog;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

注入构造函数和工厂方法参数属于特殊情况，因为@Autowired注解中的required属性含义略有不同——这源于Spring的构造函数解析算法可能处理多个构造函数。构造函数和工厂方法参数默认会被强制要求，但在单构造函数场景下存在若干特殊规则：例如当多元素注入点（数组、集合、映射）无法匹配可用Bean时，会解析为空实例。这使得一种常见实现模式成为可能：所有依赖项均可声明在单一的多参数构造器中——例如声明为不带`@Autowired`注解的公共构造器。


## 自身注入
`@Autowired` 在进行注入时也会考虑自我引用（即回指当前被注入的 Bean）。

但需注意，自我注入仅作为备用机制。对其他组件的常规依赖始终具有优先级。因此自我引用不会参与常规自动装配候选对象的选择，尤其永远不会成为首要选项，反而始终处于最低优先级。

实际应用中，仅应将自我引用作为最后手段——例如通过 bean 的事务代理调用同一实例的其他方法。在这种场景下，建议将受影响的方法提取到独立的委托 bean 中。

另一种替代方案是使用 `@Resource` 注解，它可通过唯一名称获取指向当前 bean 的代理。


## 注入集合
```
public class MovieRecommender {

	@Autowired
	private MovieCatalog[] movieCatalogs;

    @Autowire
    private Set<Movies> movies;

    @Autowire
    private Map<String, MovieCatalog> movieCatalogs;
}
```
注入数组、 `List` 、 `Set` 等集合对象时，会将指定类型的 bean 全部注入到容器中。 若需按特定顺序排列数组或列表中的元素，目标 Bean 可实现 `org.springframework.core.Ordered` 接口，或使用 `@Order` 注解及标准的 `@Priority` 注解。否则，其顺序将遵循容器中对应目标 Bean 定义在注册时的顺序。

请注意，配置类上的 `@Order` 注解仅影响启动时整个配置类顺序。此类配置级别的顺序值完全不会影响所包含的 `@Bean` 方法。对于bean级别的排序，每个 `@Bean` 方法都需要拥有自己的 `@Order` 注解，该注解仅适用于特定bean类型（由工厂方法返回）的多重匹配集合内部。

标准的 `jakarta.annotation.Priority` 注解无法在 `@Bean` 级别使用，因为它不能声明在方法上。其语义可通过在每个类型的单个 bean 上结合使用 `@Order` 值与 `@Primary` 注解来实现。

注入 `Map` 时， `key` 必须是 `String` 类型，注入的是以 `beanName` 为 key ， bean 实例为 value 的 `entry` 对象.

默认情况下，当给定注入点没有匹配的候选 Bean 时，自动装配会失败。**对于声明的数组、集合或 Map，至少需要一个匹配的元素。**


## required 属性
`@Autowired` 默认是将注解标记的方法的参数和字段视为必需依赖项。如果没有满足的依赖机会报错。 可通过将注入点标记为非必需（即在 `@Autowired` 注解中将 `required` 属性设为 `false` ）来改变此行为，从而使框架跳过无法满足的注入点。

若 `required=false` 方法的依赖项（或在存在多个参数时其中任一依赖项）不可用，**则该方法将完全不会被调用**。此类情况下，**非必需字段将完全不会被填充，其默认值将保持不变。**

换言之，将 `required` 属性设置为 `false` 表示该属性在自动装配过程中为可选项，若无法通过自动装配获取，则该属性将被忽略。这使得属性既可被赋予默认值，又可通过依赖注入进行可选覆盖。

除了使用 `@Autowired(required=false)` ，当参数类型为 `java.util.Optional<T>` 或者使用 `@Nullable` 注解标注时，也可以实现相同的效果：
```
public class SimpleMovieLister {

	@Autowired
	public void setMovieFinder(Optional<MovieFinder> movieFinder) {
		...
	}

    @Autowired
	public void setMovieFinder(@Nullable MovieFinder movieFinder) {
		...
	}
}
```

## 注入Spring提供的 bean
对于已知可解析的依赖接口，可以使用 `@Autowired` 注解： `BeanFactory` 、 `ApplicationContext` 、 `Environment` 、 `ResourceLoader` 、 `ApplicationEventPublisher` 和 `MessageSource` 。这些接口及其扩展接口（如 `ConfigurableApplicationContext` 或 `ResourcePatternResolver` ）会自动解析，无需特殊配置。以下示例演示了如何自动注入 `ApplicationContext` 对象：
```
public class MovieRecommender {

	@Autowired
	private ApplicationContext context;

	public MovieRecommender() {
	}

	// ...
}
```