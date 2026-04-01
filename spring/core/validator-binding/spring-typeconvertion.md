# Spring 类型转换
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/convert.html

`core.convert` 软件包提供了一个通用的类型转换系统。该系统定义了用于实现类型转换逻辑的 `SPI` 和用于在运行时执行类型转换的 API。在 Spring 容器中，可以使用该系统替代 `PropertyEditor` 实现，将外部化的 Bean **属性值字符串**转换为**所需的属性类型**。还可以在应用程序中需要进行类型转换的任何地方使用公共 API。

## Converter SPI
```
public interface Converter<S, T> {
	T convert(S source);
}
```
要创建自己的 `Converter` ，只需实现 `Converter` 接口，并将 `S` 设为要转换的源类型，将 `T` 设为要转换的目标类型。如果需要将 `S` 的集合或数组转换为 `T` 的数组或集合，也可以透明地应用这种 `Converter` ，前提是委托的数组或集合 `Converter` 也已注册（默认情况下， `DefaultConversionService` 已注册）。

每次调用 `convert(S)` 时，源参数保证不为 `null` 。若转换失败， `Converter` 可抛出任何 `unchecked exception` 。具体而言，应抛出 `IllegalArgumentException` 来报告无效的源值。请务必确保 `Converter` 实现是线程安全的。

核心的 `core.convert.support` 包中提供了若干转换器实现，以方便使用。包括字符串到数字及其他常见类型的转换器。以下代码展示了 `StringToInteger` 类，这是典型的 `Converter` 实现：
```
final class StringToInteger implements Converter<String, Integer> {

	public Integer convert(String source) {
		return Integer.valueOf(source);
	}
}
```

## ConverterFactory
当需要集中整个类层次结构的转换逻辑时（例如， `String` 转换为 `Enum` 对象），可以实现 `ConverterFactory` ，如下例所示：
```
public interface ConverterFactory<S, R> {
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

`S` 为要转换的类型， `R` 为定义可转换目标类范围的基础类型。然后实现 `getConverter(Class<T>)` ，其中 `T` 是 `R` 的子类。

`StringToEnumConverterFactory` 的示例：
```
final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

	public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToEnumConverter(targetType);
	}

	private final class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

		private Class<T> enumType;

		public StringToEnumConverter(Class<T> enumType) {
			this.enumType = enumType;
		}

		public T convert(String source) {
			return (T) Enum.valueOf(this.enumType, source.trim());
		}
	}
}
```

## GenericConverter
如果需要复杂的 `Converter` 实现，可以考虑使用 `GenericConverter` 接口。 `GenericConverter` 具有比 `Converter` 更灵活但强类型更少的签名，**支持在多个源和目标类型之间进行转换**。此外， `GenericConverter` 还提供源和目标字段上下文，供你在实现转换逻辑时使用。这种上下文允许通过字段注解或在字段特征上声明的通用信息来驱动类型转换。下面的列表显示了 `GenericConverter` 的接口定义：
```
public interface GenericConverter {
	public Set<ConvertiblePair> getConvertibleTypes();
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
要实现 `GenericConverter` ，可让 `getConvertibleTypes()` 返回支持的**源→目标类型对**。然后实现 `convert(Object,TypeDescriptor,TypeDescriptor)` ，以包含转换逻辑。源 `TypeDescriptor` 提供对持有被转换值的源字段的访问。目标 `TypeDescriptor` 用于访问要设置转换值的目标字段。

参照实现： `ArrayToCollectionConverter` 

## ConditionalGenericConverter
有时，我们希望 `Converter` 仅在特定条件为真时运行。例如，只想在目标字段上存在特定注解时运行 `Converter` ，或者只想在目标类上定义了特定方法（如静态 valueOf 方法）时运行 `Converter` `。ConditionalGenericConverter` 是 `GenericConverter` 和 `ConditionalConverter` 接口的结合，可让您定义此类自定义匹配条件：
```
public interface ConditionalConverter {
	boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}

public interface ConditionalGenericConverter extends GenericConverter, ConditionalConverter {
}
```

参照实现： `IdToEntityConverter` 

## ConversionService API
`ConversionService` 定义了在运行时执行类型转换逻辑的统一 API。 `Converter` 通常在这些门面接口后面运行：
```
public interface ConversionService {
	boolean canConvert(Class<?> sourceType, Class<?> targetType);
	<T> T convert(Object source, Class<T> targetType);
	boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
大多数 `ConversionService` 实现还实现了 `ConverterRegistry` ，它为注册 `Converter` 提供了SPI。在内部， `ConversionService` 实现委托其注册的 `Converter` 执行类型转换逻辑。

`core.convert.support` 包中提供了强大的 `ConversionService` 实现。 `GenericConversionService` 是适用于大多数环境的通用实现。 `ConversionServiceFactory` 提供了一个方便的工厂，用于创建常用的 `ConversionService` 配置。

### 配置 ConversionService
`ConversionService` 是一个无状态对象，设计用于在应用程序启动时实例化，然后在多个线程之间共享。在 Spring 应用程序中，通常会为每个 Spring 容器（或 ApplicationContext）配置一个 `ConversionService` 实例。只要框架需要执行类型转换，Spring 就会获取并使用该 `ConversionService` 。还可以将此 `ConversionService` 注入到您的任何 Bean 中，并直接调用它。

要向 Spring 注册默认 ConversionService，请添加以下 Bean 定义，id 为 conversionService：
```
<bean id="conversionService"
	class="org.springframework.context.support.ConversionServiceFactoryBean"/>
```
默认 `ConversionService` 可以在字符串、数字、枚举、集合、MAp和其他常见类型之间进行转换。要使用自定义 `Converter` 补充或覆盖默认 `Converter` ，请设置 `converters` 属性。属性值可以实现任何 `Converter` 、 `ConverterFactory` 或 `GenericConverter` 接口。
```
<bean id="conversionService"
		class="org.springframework.context.support.ConversionServiceFactoryBean">
	<property name="converters">
		<set>
			<bean class="example.MyCustomConverter"/>
		</set>
	</property>
</bean>
```

### 以编程方式使用 ConversionService
要以编程方式使用 `ConversionService` 实例，可以像使用其他 Bean 一样注入一个 `ConversionService` 。下面的示例展示了如何做到这一点：
```
@Service
public class MyService {
	private final ConversionService conversionService;

	public MyService(ConversionService conversionService) {
		this.conversionService = conversionService;
	}

	public void doIt() {
		this.conversionService.convert(...)
	}
}
```

对于大多数使用场景，您可以使用 `targetType` 类型的转换方法，但该方法不适用于更复杂的类型，例如参数化元素的集合。若要通过编程方式将 `List<Integer>` 转换为 `List<String>` ，则需要提供源类型和目标类型的正式定义。

`TypeDescriptor` 可以做到这一点：

```
DefaultConversionService cs = new DefaultConversionService();
List<Integer> input = ...
cs.convert(input,
	TypeDescriptor.forObject(input), // List<Integer> type descriptor
	TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(String.class)));
```

请注意， `DefaultConversionService` 会自动注册适用于大多数环境的转换器。这包括 `collection converters` , `scalar converters` , and basic `Object-to-String converters`  。 还可以通过调用 `DefaultConversionService` 类的静态 `addDefaultConverters` 方法，将相同的转换器注册到任何 `ConverterRegistry` 中。

值类型的 `Converter` 可重复用于数组和集合，因此无需创建特定的 `Converter` 来将 `S` 的集合转换为 `T` 的集合。  
也就是说如果有一个 `Converter` 能将 `S` 转换为 `T` , 那么无需在创建一个 `Converter` 来将 `List<S>` 转换为 `List<T` .