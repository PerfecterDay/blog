# @Value
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/value-annotations.html


`@Value` 通常用于注入外部化的属性值：
```
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("${catalog.name}") String catalog) {
		this.catalog = catalog;
	}
}
```


Spring 默认提供了一个宽松的嵌入值解析器。它会尝试解析属性值，若解析失败，则将属性名称（例如 `${catalog.name}`）作为值注入。若需对不存在的值实施严格控制，应声明一个 `PropertySourcesPlaceholderConfigurer` Bean，如下例所示：
```
@Configuration
public class AppConfig {

	@Bean
	public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
		return new PropertySourcesPlaceholderConfigurer();
	}
}
```

使用 JavaConfig 配置 `PropertySourcesPlaceholderConfigurer` 时， `@Bean` 方法必须为 `static` 方法。

使用上述配置可确保当任何 `${}` 占位符无法解析时，Spring 初始化将失败。也可通过 `setPlaceholderPrefix()` 、 `setPlaceholderSuffix()` 、`setValueSeparator()` 或 `setEscapeCharacter()` 等方法自定义占位符语法。此外，可通过JVM系统属性（或SpringProperties机制）设置 `spring.placeholder.escapeCharacter.default` 属性，实现全局范围内的默认转义字符修改或禁用。

Spring Boot 默认配置了一个 `PropertySourcesPlaceholderConfigurer`  bean，该 bean 将从 `application.properties` 和 `application.yml` 文件中获取属性。

Spring提供的内置转换器支持可自动处理简单的类型转换（例如转换为 `Integer` 或 `int` ）。多个以逗号分隔的值可自动转换为字符串数组，无需额外操作。

还可以指定默认值：
```
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
		this.catalog = catalog;
	}
}
```

Spring `BeanPostProcessor` 在后台使用 `ConversionService` 来处理将 `@Value` 中字符串值转换为目标类型的过程。若需为自定义类型提供转换支持，可按以下示例提供自定义的 `ConversionService` Bean 实例：
```
@Configuration
public class AppConfig {

	@Bean
	public ConversionService conversionService() {
		DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
		conversionService.addConverter(new MyCustomConverter());
		return conversionService;
	}
}
```

当 `@Value` 包含 `SpEL` 表达式时，其值将在运行时动态计算，如下例所示：
```
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
		this.catalog = catalog;
	}
}
```

`SpEL` 甚至可以包含复杂的数据结构：
```
@Component
public class MovieRecommender {

	private final Map<String, Integer> countOfMoviesPerCatalog;

	public MovieRecommender(
			@Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
		this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
	}
}
```