# BeanDefinition中的SpEL表达式
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/expressions/beandef.html

可以使用 SpEL 表达式配合配置元数据来定义 Bean 实例。在两种情况下，定义表达式的语法形式均为 `#{ <表达式字符串> }`。

应用程序上下文中的所有 Bean 都可作为**预定义变量使用，其名称即为通用 Bean 名称**。这包括标准上下文 Bean（如类型为 `org.springframework.core.env.Environment` 的环境 Bean），以及用于访问运行时环境的 `systemProperties` 和 `systemEnvironment` （类型为 `Map<String, Object>` ）。

要指定默认值，可以在字段、方法以及方法或构造函数参数上添加 `@Value` 注解（或使用XML等效标记）。

1. 注解再字段上：
```
public class FieldValueTestBean {

	@Value("#{ systemProperties['user.region'] }")
	private String defaultLocale;

	public void setDefaultLocale(String defaultLocale) {
		this.defaultLocale = defaultLocale;
	}

	public String getDefaultLocale() {
		return this.defaultLocale;
	}
}
```

2. 注解在 `setter` 方法上：
```
public class PropertyValueTestBean {

	private String defaultLocale;

	@Value("#{ systemProperties['user.region'] }")
	public void setDefaultLocale(String defaultLocale) {
		this.defaultLocale = defaultLocale;
	}

	public String getDefaultLocale() {
		return this.defaultLocale;
	}
}
```

3. 注解在构造函数上：
```
public class SimpleMovieLister {

	private MovieFinder movieFinder;
	private String defaultLocale;

	@Autowired
	public void configure(MovieFinder movieFinder,
			@Value("#{ systemProperties['user.region'] }") String defaultLocale) {
		this.movieFinder = movieFinder;
		this.defaultLocale = defaultLocale;
	}

    public SimpleMovieLister(MovieFinder movieFinder,
			@Value("#{systemProperties['user.country']}") String defaultLocale) {
		this.movieFinder = movieFinder;
		this.defaultLocale = defaultLocale;
	}

	// ...
}
```

4. 通过名称引用其他 Bean 属性，如下例所示：
```
public class ShapeGuess {

	private double initialShapeSeed;

	@Value("#{ numberGuess.randomNumber }")
	public void setInitialShapeSeed(double initialShapeSeed) {
		this.initialShapeSeed = initialShapeSeed;
	}

	public double getInitialShapeSeed() {
		return initialShapeSeed;
	}
}
```