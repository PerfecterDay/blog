# @Resource 注解
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/resource.html

Spring 还支持通过在字段或 Bean `setter` 方法上使用 JSR-250 `@Resource` 注解（ `jakarta.annotation.Resource` ）实现依赖注入。

`@Resource` 注解接受一个 `name` 属性。默认情况下，Spring将该值解释为待注入的**bean名称**。换言之，它遵循按名称注入的语义，如下例所示：
```
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource(name="myMovieFinder")
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

若未显式指定名称，则默认名称由字段名或设置器方法名推导而来。对于字段，采用字段名；对于设置器方法，采用 Bean 属性名。以下示例将把名为 movieFinder 的 Bean 注入其设置器方法：
```
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```
在仅使用@ `Resource` 且未显式指定名称的特殊情况下，与 `@Autowired` 类似， `@Resource` 会寻找主要类型匹配而非特定命名的Bean，并解析以下已知可解析的依赖项： `BeanFactory` 、 `ApplicationContext` 、 `ResourceLoader` 、 `ApplicationEventPublisher` 和 `MessageSource` 接口。

因此，在下面的示例中，customerPreferenceDao 字段首先查找名为 "customerPreferenceDao" 的 Bean，然后回退到对 CustomerPreferenceDao 类型的主类型匹配：
```
public class MovieRecommender {

	@Resource
	private CustomerPreferenceDao customerPreferenceDao;

	@Resource
	private ApplicationContext context;

	public MovieRecommender() {
	}

	// ...
}
```

`CommonAnnotationBeanPostProcessor` 负责处理 `@Resource` 注解，此外， JSR-250 的其它注解： `@PostConstruct` 和 `@PreDestroy` 也是由它处理。