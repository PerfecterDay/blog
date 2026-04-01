# Spring 全局日期和时间配置
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/format-configuring-formatting-globaldatetimeformat.html

默认情况下，未标注 `@DateTimeFormat` 的日期和时间字段将使用 `DateFormat.SHORT` 样式从字符串转换而来。若需调整，可通过定义全局格式进行修改。

要实现这一点，请确保Spring不会注册默认格式化器。相反，请通过以下方式手动注册格式化器：
+ `org.springframework.format.datetime.standard.DateTimeFormatterRegistrar`
+ `org.springframework.format.datetime.DateFormatterRegistrar`

示例：
```
@Configuration
public class ApplicationConfiguration {

	@Bean
	public FormattingConversionService conversionService() {

		// Use the DefaultFormattingConversionService but do not register defaults
		DefaultFormattingConversionService conversionService =
				new DefaultFormattingConversionService(false);

		// Ensure @NumberFormat is still supported
		conversionService.addFormatterForFieldAnnotation(
				new NumberFormatAnnotationFormatterFactory());

		// Register JSR-310 date conversion with a specific global format
		DateTimeFormatterRegistrar dateTimeRegistrar = new DateTimeFormatterRegistrar();
		dateTimeRegistrar.setDateFormatter(DateTimeFormatter.ofPattern("yyyyMMdd"));
		dateTimeRegistrar.registerFormatters(conversionService);

		// Register date conversion with a specific global format
		DateFormatterRegistrar dateRegistrar = new DateFormatterRegistrar();
		dateRegistrar.setFormatter(new DateFormatter("yyyyMMdd"));
		dateRegistrar.registerFormatters(conversionService);

		return conversionService;
	}
}
```

请注意，如果是在 Web 环境下配置全局的日期和时间格式化，需要额外的步骤。

