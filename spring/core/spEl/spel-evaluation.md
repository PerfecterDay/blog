# Evaluation
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html

`Hello World` 示例:
```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'");
String message = (String) exp.getValue();
```

最常使用的SpEL类和接口位于 `org.springframework.expression` 包及其子包中，例如 `org.springframework.expression.spel.support` 。

`ExpressionParser` 接口负责解析表达式字符串。在上例中，表达式字符串是由单引号包围的字符串字面量。 `Expression` 接口负责评估计算定义的表达式字符串。调用  `parser.parseExpression(…​)` 和 `exp.getValue(…​)` 时可能分别抛出的两种异常: `ParseException` 和 `EvaluationException` 。

SpEL支持多种功能，包括调用方法、访问属性以及调用构造函数。
```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'.concat('!')");
String message = (String) exp.getValue();

// invokes 'getBytes()'
Expression exp = parser.parseExpression("'Hello World'.bytes");
byte[] bytes = (byte[]) exp.getValue();
```


SpEL还支持使用标准点表示法（如 `prop1.prop2.prop3` ）访问嵌套属性，也可以设置嵌套属性值。公共字段同样可被访问。
```
ExpressionParser parser = new SpelExpressionParser();

// invokes 'getBytes().length'
Expression exp = parser.parseExpression("'Hello World'.bytes.length");
int length = (Integer) exp.getValue();
```

字符串的构造函数可替代字符串字面量使用，如下例所示：
```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("new String('hello world').toUpperCase()");
String message = exp.getValue(String.class);
```
注意，如果想构造一个自定义的类型，必须将类型定义为 `public class`，并且指定包含包路径的全限定名：
```
@Data
public class Person{
    int age;
    String name;
}

Expression expression = parser.parseExpression("new spring.spel.Person()");
```

请注意泛型方法的使用： `public <T> T getValue(Class<T> desiredResultType)` 。使用此方法可免去将表达式值强制转换为预期结果类型的步骤。若值无法转换为类型 `T` 或通过注册的类型转换器进行转换，则会抛出 `EvaluationException` 。

SpEL更常见的用法是提供一个表达式字符串，该字符串将针对特定对象实例（称为根对象）进行评估。以下示例演示了如何从 `Inventor` 类的实例中获取 `name` 属性，以及如何在布尔表达式中引用该属性。
```
// Create and set a calendar
GregorianCalendar c = new GregorianCalendar();
c.set(1856, 7, 9);

// The constructor arguments are name, birthday, and nationality.
Inventor tesla = new Inventor("Nikola Tesla", c.getTime(), "Serbian");

ExpressionParser parser = new SpelExpressionParser();

Expression exp = parser.parseExpression("name"); // Parse name as an expression
String name = (String) exp.getValue(tesla);
// name == "Nikola Tesla"

exp = parser.parseExpression("name == 'Nikola Tesla'");
boolean result = exp.getValue(tesla, Boolean.class);
// result == true
```


## EvaluationContext
在评估表达式时， `EvaluationContext` API 用于解析属性、方法或字段，并协助执行类型转换。Spring 提供了两种实现方案。

+ `SimpleEvaluationContext`: 暴露了SpEL语言中一组核心功能和配置选项，适用于无需完整语法支持且应合理限制的表达式类别。例如数据绑定表达式和基于属性的过滤器等（但不限于此）。
+ `StandardEvaluationContext`: 全面包含 SpEL 的语言特性和配置选项。可以使用它来指定默认根对象，并配置所有可用的评估相关策略。

值得一提的是， `SimpleEvaluationContext.forReadOnlyDataBinding()` 可通过 `DataBindingPropertyAccessor` 实现对属性的只读访问。同样地，`SimpleEvaluationContext.forReadWriteDataBinding()` 则支持对属性的读写操作。此外，还可通过 `SimpleEvaluationContext.forPropertyAccessors(…)` 配置自定义访问器，可选择禁用赋值操作，并可通过构建器选项激活方法解析和/或类型转换器。

### 类型转换
默认情况下，SpEL 使用 Spring 核心提供的类型转换服务（ `org.springframework.core.convert.ConversionService` ）。该转换服务内置了多种常用转换器，同时支持完全扩展，允许用户添加自定义类型转换。此外，它还具备泛型感知能力。这意味着当表达式中涉及泛型类型时，SpEL 会尝试进行转换以确保遇到任何对象时都能保持类型正确性。

这在实际中意味着什么？假设使用 `setValue()` 赋值来设置一个 `List` 属性。该属性的实际类型是 `List<Boolean>` 。SpEL 会识别到列表元素需要转换为布尔值后才能放入其中。以下示例展示了如何实现：
```
class Simple {
	public List<Boolean> booleanList = new ArrayList<>();
}

Simple simple = new Simple();
simple.booleanList.add(true);

EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

// "false" is passed in here as a String. SpEL and the conversion service
// will recognize that it needs to be a Boolean and convert it accordingly.
parser.parseExpression("booleanList[0]").setValue(context, simple, "false");

// b is false
Boolean b = simple.booleanList.get(0);
```


## Parser 配置
可以通过解析器配置对象（ `org.springframework.expression.spel.SpelParserConfiguration` ）来配置SpEL表达式解析器 `ExpressionParser` 。该配置对象可控制某些表达式组件的行为。例如，当对集合进行索引操作时，若指定索引处的元素为空，SpEL可自动创建该元素。此特性在处理由属性引用链组成的表达式时尤为实用。同样地，若对集合进行索引操作时指定的索引超出当前集合大小，SpEL会自动扩展集合以容纳该索引。为在指定索引处添加元素时，SpEL会先尝试使用元素类型的默认构造函数创建该元素，再设置指定值。若元素类型无默认构造函数，则向集合添加 `null` 值。若无内置转换器或自定义转换器能处理该值，则指定索引处的 `null` 值将保留在集合中。以下示例演示了如何自动扩展 `List` 集合：
```
class Demo {
	public List<String> list;
}

// Turn on:
// - auto null reference initialization
// - auto collection growing
SpelParserConfiguration config = new SpelParserConfiguration(true, true);

ExpressionParser parser = new SpelExpressionParser(config);

Expression expression = parser.parseExpression("list[3]");

Demo demo = new Demo();

Object o = expression.getValue(demo);

// demo.list will now be a real collection of 4 entries
// Each entry is a new empty String
```

默认情况下，SpEL表达式不能超过 `10,000` 个字符，但 `maxExpressionLength` 是可配置的。若通过编程方式创建 `SpelExpressionParser` ，可在创建提供给 `SpelExpressionParser` 的 `SpelParserConfiguration` 时指定自定义的 `maxExpressionLength` 。若需设置应用程序上下文（例如XML bean定义、 `@Value` 注解等场景）中解析SpEL表达式的最大长度，可通过JVM系统属性或Spring配置： `spring.context.expression.maxLength` 配置所需的最大表达式长度。


## 编译 SpEL
Spring 为 SpEL 表达式提供了一个基础编译器。表达式通常采用**解释执行**方式，这在评估运行过程中提供了高度的动态灵活性，但无法实现最佳性能。对于偶尔使用的表达式而言，这种方式尚可接受；然而当被 Spring Integration 等其他组件调用时，性能就变得至关重要，此时动态特性已无实际必要。

SpEL编译器的设计正是为满足这一需求。在表达式求值过程中，编译器会生成一个Java类来体现运行时的表达式行为，并利用该类实现更快速的表达式求值。由于表达式本身缺乏类型信息，编译器在编译时会利用表达式在解释性求值过程中收集信息。例如，仅凭表达式本身无法确定属性引用的类型，但通过首次解释性求值即可确定其类型。当然，基于此类推导信息进行编译可能引发后续问题——若表达式各元素的类型随时间变化，便会导致冲突。因此，编译机制最适用于**类型信息在重复评估中保持不变**的表达式。

```
someArray[0].someProperty.someOtherProperty < 0.1
```

上述的表达式涉及数组访问、属性解引用和数值运算，性能提升效果非常显著。在一次包含50,000次迭代的微基准测试中，使用解释器评估该表达式耗时**75毫秒**，而使用编译版本仅需**3毫秒**。

### 编译器配置
编译器默认处于关闭状态，但您可通过两种方式启用它：
+ 一是通过解析器配置流程（前文已述）
+ 二是当SpEL嵌入其他组件时使用Spring属性。

编译器可运行于三种模式之一，这些模式通过 `org.springframework.expression.spel.SpelCompilerMode` 枚举进行定义。具体模式如下：

+ `OFF` ：编译器关闭，所有表达式都将以解释模式进行评估运行。这是默认模式。
+ `IMMEDIATE` : 在此模式下，表达式会在尽可能早的时候进行编译，通常是在首次解释性评估之后。若编译后的表达式评估失败（例如因类型变更导致，如前所述），则表达式评估的调用方将收到异常。若表达式中各元素的类型随时间变化，请考虑切换至混合模式或关闭编译器。
+ `MIXED` : 在混合模式下，表达式评估会随时间在解释模式与编译模式间无声切换。经过若干次成功的解释运行后，该表达式将被编译。若编译后表达式的评估失败（例如因类型变更），**系统将内部捕获该失败并切换回该表达式的解释模式**。本质上，调用者在 `IMMEDIATE` 模式下收到的异常将转为内部处理。随后编译器可能生成新的编译形式并切换至该模式。这种解释与编译模式的循环切换将持续进行，直至系统判定继续尝试已无意义——例如达到特定失败阈值时——此时系统将永久切换为该表达式的解释模式。

**`IMMEDIATE` 模式的存在是因为 `MIXED` 模式可能导致具有副作用的表达式出现问题。如果编译后的表达式在部分成功后发生异常，它可能已经执行了某些影响系统状态的操作。若发生这种情况，调用方可能不希望该表达式在解释模式下静默重试，因为表达式的一部分可能会被重复执行从而带来非预期的结果。**

选择模式后，使用 `SpelParserConfiguration` 配置解析器。以下示例展示了具体操作方法。
```
SpelParserConfiguration config = new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE,
		this.getClass().getClassLoader());

SpelExpressionParser parser = new SpelExpressionParser(config);
Expression expr = parser.parseExpression("payload");
MyMessage message = new MyMessage();
Object payload = expr.getValue(message);
```

指定编译器模式时，也可指定 `ClassLoader`（允许传入 `null` ）。编译后的表达式将定义在所提供 `ClassLoader` 下的子 `ClassLoader` 中。需确保指定的 `ClassLoader` 能访问表达式评估过程中涉及的所有类型。若未指定 `ClassLoader` ，则使用默认 `ClassLoader` （通常为表达式评估期间运行线程的上下文 `ClassLoader` ）。

配置编译器的第二种方式适用于SpEL嵌入在其他组件内部的情形，此时可能无法通过配置对象进行配置。在这种情况下，可通过JVM系统属性（或SpringProperties机制）将 `spring.expression.compiler.mode` 属性设置为 `SpelCompilerMode` 枚举值之一（ `off` 、 `immediate` 或 `mixed` ）。

### 编译器的局限性
Spring 并不支持编译所有类型的表达式。其主要关注点在于那些可能在性能关键场景中使用的常见表达式。以下类型的表达式无法被编译：
+ Expressions involving assignment
+ Expressions relying on the conversion service
+ Expressions using custom resolvers
+ Expressions using overloaded operators
+ Expressions using array construction syntax
+ Expressions using selection or projection
+ Expressions using bean references