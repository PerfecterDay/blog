# @Bean 注解
{docsify-updated}

### 定义 bean 的名称
默认情况下，配置类会将 `@Bean` 注解**方法的名称**作为生成的Bean名称。不过，此功能可通过 `name` 属性进行覆盖，如下例所示：
```
@Configuration
public class AppConfig {

	@Bean("myThing")
	public Thing thing() {
		return new Thing();
	}
}
```

### Bean Aliasing
```
@Configuration
public class AppConfig {

	@Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
	public DataSource dataSource() {
		// instantiate, configure and return DataSource bean...
	}
}
```

### Bean Description
```
@Configuration
public class AppConfig {

	@Bean
	@Description("Provides a basic example of a bean")
	public Thing thing() {
		return new Thing();
	}
}
```