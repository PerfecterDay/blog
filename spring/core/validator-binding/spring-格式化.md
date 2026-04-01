# Spring 类型转换与格式化
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/format.html

总的来说， `Formatter` 主要用于用户界面的数据格式化和解析，通常是用于针对单个字段或特定类型的格式化和解析，比如日期、货币等。而 `ConversionService` 则用于更广泛的**类型转换**，涵盖了整个应用程序的类型转换需求。

## The Formatter SPI
```
package org.springframework.format;

public interface Formatter<T> extends Printer<T>, Parser<T> {
}

public interface Printer<T> {
	String print(T fieldValue, Locale locale); ////Java对象转换为显示数据
}

public interface Parser<T> {
	T parse(String clientValue, Locale locale) throws ParseException; //显示数据转换为Java对象
}
```
要实现自定义的格式化 ，实现 `Formatter` 接口即可。将 T 设为要格式化的对象类型，例如 `java.util.Date` 。执行 `print()` 操作来打印 T 的实例，以便在客户端本地显示。执行 `parse()` 操作，从客户端本地语言返回的格式化表示中解析 T 的实例。如果解析尝试失败，格式器应抛出 `ParseException` 或 `IllegalArgumentException` 异常。请注意确保您的 Formatter 实现是线程安全的。

```
package org.springframework.format.datetime;

public final class DateFormatter implements Formatter<Date> {

	private String pattern;

	public DateFormatter(String pattern) {
		this.pattern = pattern;
	}

	public String print(Date date, Locale locale) {
		if (date == null) {
			return "";
		}
		return getDateFormat(locale).format(date);
	}

	public Date parse(String formatted, Locale locale) throws ParseException {
		if (formatted.length() == 0) {
			return null;
		}
		return getDateFormat(locale).parse(formatted);
	}

	protected DateFormat getDateFormat(Locale locale) {
		DateFormat dateFormat = new SimpleDateFormat(this.pattern, locale);
		dateFormat.setLenient(false);
		return dateFormat;
	}
}
```


## 注解驱动的格式化
字段格式可按字段类型或注解配置。要将注解绑定到格式器，需要实现 `AnnotationFormatterFactory` 。下表显示了 AnnotationFormatterFactory 接口的定义：
```
package org.springframework.format;

public interface AnnotationFormatterFactory<A extends Annotation> {
	Set<Class<?>> getFieldTypes();
	Printer<?> getPrinter(A annotation, Class<?> fieldType);
	Parser<?> getParser(A annotation, Class<?> fieldType);
}
```
实现一个 `AnnotationFormatterFactory` 的步骤：
1. 将 A 设为要关联格式化逻辑的字段**注解**类型参数，例如 `org.springframework.format.annotation.DateTimeFormat` 。
2. 让 `getFieldTypes()` 返回可使用该注解的字段类型。
3. 让 `getPrinter()` 返回 `Printer` 将注解字段Java对象输出为格式化输出。
4. 让 `getParser()` 返回一个 `Parser` ，用于解析注解字段值到Java对象。

下面的示例 `AnnotationFormatterFactory` 实现将 `@NumberFormat` 注解与 `Formatter` 绑定，以便指定数字样式或模式：
```
public final class NumberFormatAnnotationFormatterFactory
		implements AnnotationFormatterFactory<NumberFormat> {

	private static final Set<Class<?>> FIELD_TYPES = Set.of(Short.class,
			Integer.class, Long.class, Float.class, Double.class,
			BigDecimal.class, BigInteger.class);

	public Set<Class<?>> getFieldTypes() {
		return FIELD_TYPES;
	}

	public Printer<Number> getPrinter(NumberFormat annotation, Class<?> fieldType) {
		return configureFormatterFrom(annotation, fieldType);
	}

	public Parser<Number> getParser(NumberFormat annotation, Class<?> fieldType) {
		return configureFormatterFrom(annotation, fieldType);
	}

	private Formatter<Number> configureFormatterFrom(NumberFormat annotation, Class<?> fieldType) {
		if (!annotation.pattern().isEmpty()) {
			return new NumberStyleFormatter(annotation.pattern());
		}
		// else
		return switch(annotation.style()) {
			case Style.PERCENT -> new PercentStyleFormatter();
			case Style.CURRENCY -> new CurrencyStyleFormatter();
			default -> new NumberStyleFormatter();
		};
	}
}
```

使用注解：
```
public class MyModel {
	@NumberFormat(style=Style.CURRENCY)
	private BigDecimal decimal;

	@DateTimeFormat(iso=ISO.DATE)
	private Date date;
}
```

### Format Annotation API


## FormatterRegistry SPI
`FormatterRegistry` 是用于注册格式器和转换器的 SPI。 `FormattingConversionService` 是 `FormatterRegistry` 的一种实现，适用于大多数环境。以通过编程或声明的方式将此变体配置为 Spring Bean，例如使用 `FormattingConversionServiceFactoryBean` 。由于该实现也实现了 `ConversionService` ，因此可以直接将其配置为与 Spring 的 `DataBinder` 和 Spring Expression Language (SpEL) 一起使用。
```
package org.springframework.format;

public interface FormatterRegistry extends ConverterRegistry {
	void addPrinter(Printer<?> printer);
	void addParser(Parser<?> parser);
	void addFormatter(Formatter<?> formatter);
	void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);
	void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);
	void addFormatterForFieldAnnotation(AnnotationFormatterFactory<? extends Annotation> annotationFormatterFactory);
}
```
可以按字段类型或注解注册格式器。 `FormatterRegistry` SPI 可让您集中配置格式化规则，而无需在各个控制器中重复配置。例如，如果希望强制所有日期字段以某种方式格式化，或强制具有特定注解的字段以某种方式格式化。有了共享的 `FormatterRegistry` ，您只需定义一次这些规则，它们就会在需要格式化时被应用。

## FormatterRegistrar SPI
```
package org.springframework.format;

public interface FormatterRegistrar {
	void registerFormatters(FormatterRegistry registry);
}
```
`FormatterRegistrar` 适用于为特定格式类别（如日期格式化）注册多个相关的 converters 和 formatters 。当声明式注册不够充分时，例如，当 formatter 需要在不同于其自身 <T> 的特定字段类型下进行索引时，或者当注册 `Printer/Parser` 。

## 全局配置日期和时间格式化
默认情况下，未注解 `@DateTimeFormat` 的日期和时间字段会使用 `DateFormat.SHORT` 样式转换为字符串。可以通过定义自己的全局格式来更改这个默认行为。
```
@Configuration
public class AppConfig {

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
如果是 Spring Web 环境：
```
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

	@Override
	public void addFormatters(FormatterRegistry registry) {
		DateTimeFormatterRegistrar registrar = new DateTimeFormatterRegistrar();
		registrar.setUseIsoFormat(true);
		registrar.registerFormatters(registry);
     	}
}
```


