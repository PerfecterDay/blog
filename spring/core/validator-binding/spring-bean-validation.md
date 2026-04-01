# Java Bean Validation
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html

Bean Validation 通过约束声明和元数据为 Java 应用程序提供了一种通用的验证方式。使用时，只需在 POJO bean 的属性上添加声明式验证约束注解，这些约束将被 Bean Validation Provider 强制执行。系统内置了多种约束，也可以定义自定义约束。

一个普通的 POJO：
```
public class PersonForm {
	private String name;
	private int age;
}
```

使用 Bean Validation 后：
```
public class PersonForm {

	@NotNull
	@Size(max=64)
	private String name;

	@Min(0)
	private int age;
}
```

Bean Validation 验证器随后会根据声明的约束条件对该类的实例进行验证。 `Jakarta Validation` 定义了一套关于验证相关的规范， `Hibernate Validator` 是这套规范的一个实现。 

## Jakarta Bean Validation
`Jakarta Bean Validation` 是Java为Bean数据合法性校验提供的标准框架，它定义了一套可标注在成员变量，属性方法上的校验注解。是一套JSR标准。  
`Hibernate Validation` 提供了这套标准的实现，在我们引入 `Spring boot starter validation` 的时候，默认会引入 `Hibernate Validation` 。

下边是一个使用 `Jakarta Bean Validation`（ `Hibernate Validator` 实现）的示例：
```
@Data
class Person{
    @NotBlank
    @NotNull
    private String name;
    @Min(200)
    private int age;

    public static void main(String[] args) {
        ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
                .configure()
                .buildValidatorFactory();
        javax.validation.Validator validator = validatorFactory.getValidator();

        Person p = new Person();
        p.setAge(120);
        Set<ConstraintViolation<Person>> constraintViolations =
                validator.validate( p );
        System.out.println(constraintViolations.size());
    }
}
```
<center><img src="pics/hibernate-validator.jpg" alt=""></center>

### Hibernate Validator 的SPI 机制
<center><img src="pics/hibernate-validator-spi.png" alt=""></center>

### 常见的Jakarta校验注解
Jakarta Bean Validation 提供的标准注解：
+ `@Null` 被注释的元素必须为 null
+ `@NotNull` 被注释的元素必须不为 null
+ `@AssertTrue` 被注释的元素必须为 true
+ `@AssertFalse` 被注释的元素必须为 false
+ `@Min(value)` 被注释的元素必须是一个数字，其值必须大于等于指定的最小值
+ `@Max(value)` 被注释的元素必须是一个数字，其值必须小于等于指定的最大值
+ `@DecimalMin(value)` 被注释的元素必须是一个数字，其值必须大于等于指定的最小值
+ `@DecimalMax(value)` 被注释的元素必须是一个数字，其值必须小于等于指定的最大值
+ `@Size(max=, min=)` 被注释的元素的大小必须在指定的范围内
+ `@Digits (integer, fraction)` 被注释的元素必须是一个数字，其值必须在可接受的范围内
+ `@Past` 被注释的元素必须是一个过去的日期
+ `@Future` 被注释的元素必须是一个将来的日期
+ `@Pattern(regex=,flag=)` 被注释的元素必须符合指定的正则表达式

Hibernate Validator提供的校验注解：
+ `@NotBlank(message =)` 验证字符串非null，且长度必须大于0
+ `@Email` 被注释的元素必须是电子邮箱地址
+ `@Length(min=,max=)` 被注释的字符串的大小必须在指定的范围内
+ `@NotEmpty` 被注释的字符串的必须非空
+ `@Range(min=,max=,message=)` 被注释的元素必须在合适的范围内

## 配置 Bean Validation Provider
Spring 完全支持 Bean Validation API，包括引导加载 Bean Validation Provider （如 `Hibernate Validator` ）注册为 Spring Bean。这使得能够在应用程序中需要验证的任何位置注入 `jakarta.validation.ValidatorFactory` 或 `jakarta.validation.Validator` 。

可以将 `LocalValidatorFactoryBean` 配置为默认的 `Validator` :
```
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;

@Configuration
public class AppConfig {

	@Bean
	public LocalValidatorFactoryBean validator() {
		return new LocalValidatorFactoryBean();
	}
}
```
前例中的基本配置会通过默认引导机制初始化 Bean Validation 。系统期望类路径中存在 Bean Validation Provider 提供者（如 Hibernate Validator），并会自动检测到该提供者。

`LocalValidatorFactoryBean` 同时实现了 `jakarta.validation.Validator` 、`jakarta.validation.ValidatorFactory`，同时还适配了 `org.springframework.validation.Validator`， 所以不管是想直接使用 `jakarta.validation.Validator` 或是使用 Spring 自定义的 `org.springframework.validation.Validator` ，直接注入对应类型的 bean 即可：
```
@Service
public class MyService {
	@Autowired
	private jakarta.validation.Validator validator;
	@Autowired
	private org.springframework.validation.Validator validator;
}
```

当作为 `org.springframework.validation.Validator` 使用时， `LocalValidatorFactoryBean` 会调用底层的 `jakarta.validation.Validator` ，随后将 `ConstraintViolations` 转换为 `FieldErrors` ，并将其注册到传递给 `validate` 方法的 `Errors` 对象中。

### 自定义校验注解
有时候，第三方库中并没有我们想要的校验类型，好在系统提供了很好的扩展能力，我们可以自定义检验。  
比如，我们想校验用户的手机格式，写手机号码校验器

1、编写校验注解
```
// 我们可以直接拷贝系统内的注解如@Min，复制到我们新的注解中，然后根据需要修改。
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
//注解的实现类。
@Constraint(validatedBy = {IsMobileValidator.class})
public @interface IsMobile {
 //校验错误的默认信息
 String message() default "手机号码格式有问题";
 //是否强制校验
 boolean isRequired() default false;
 Class<?>[] groups() default {};
 Class<? extends Payload>[] payload() default {};
}
```
`message/groups/payload` 这些成员必须要有。

2、编写具体的实现类

注解只是一个标记，真正的逻辑还要在特定的类中实现，上一步的注解指定了实现校验功能的类为 `IsMobileValidator` 。
```
// 自定义注解一定要实现ConstraintValidator接口奥，里面的两个参数
// 第一个为 具体要校验的注解
// 第二个为 校验的参数类型
public class IsMobileValidator implements ConstraintValidator<IsMobile, String> {

 @Autowired;
 private Foo aDependency;
 private boolean required = false;

 private static final Pattern mobile_pattern = Pattern.compile("1\\d{10}");
 //工具方法，判断是否是手机号
 public static boolean isMobile(String src) {
  if (StringUtils.isEmpty(src)) {
   return false;
  }
  Matcher m = mobile_pattern.matcher(src);
  return m.matches();
 }

 @Override
 public void initialize(IsMobile constraintAnnotation) {
  required = constraintAnnotation.isRequired();
 }

 @Override
 public boolean isValid(String phone, ConstraintValidatorContext constraintValidatorContext) {
  //是否为手机号的实现
  if (required) {
   return isMobile(phone);
  } else {
   if (StringUtils.isEmpty(phone)) {
    return true;
   } else {
    return isMobile(phone);
   }
  }
 }
}
```

为将声明与实现关联，每个 `@Constraint` 注解都会引用对应的 `ConstraintValidator` 实现类。运行时，当 POJO 中使用了自定义约束注解时， `ConstraintValidatorFactory` 会实例化所引用的 `ConstraintValidator` 实现类。

默认情况下， `LocalValidatorFactoryBean` 会配置一个 `SpringConstraintValidatorFactory` ，该工厂利用 Spring 创建 `ConstraintValidator` 实例。这使得自定义的 `ConstraintValidator` 能像其他 Spring Bean 一样受益于依赖注入机制。

注意： `ConstraintValidator` 实现类**必须是 public 的**。

```
import jakarta.validation.ConstraintValidator;

public class MyConstraintValidator implements ConstraintValidator {

	@Autowired;
	private Foo aDependency;

	// ...
}
```

## 方法验证
可以通过定义 `MethodValidationPostProcessor` Bean，将 Bean Validation 机制用于方法参数的验证：
```
@Configuration
public class ApplicationConfiguration {

	@Bean
	public static MethodValidationPostProcessor validationPostProcessor() {
		return new MethodValidationPostProcessor();
	}
}
```

要使方法符合Spring驱动的方法验证条件，目标类需要添加Spring的 `@Validated` 注解，该注解还可选地声明要使用的验证组。

方法验证依赖于AOP代理，包括用于接口方法的JDK动态代理或CGLIB代理。使用代理存在某些限制，其中部分限制已在[理解AOP代理](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html#aop-understanding-aop-proxies)中描述。此外请务必始终使用代理类的方法和访问器；直接访问字段将无法正常工作。

Spring MVC 和 WebFlux 内置了对相同底层方法验证的支持，且无需使用面向切面编程（AOP）。

### 方法验证异常
默认情况下，当 `jakarta.validation.Validator` 返回 `ConstraintViolations` 集合时，会抛出 `jakarta.validation.ConstraintViolationException` 。作为替代方案，您可以改为抛出 `MethodValidationException` 以适配 `MessageSourceResolvable` 错误处理。要启用此功能，请设置以下标志：
```
@Configuration
public class ApplicationConfiguration {

	@Bean
	public static MethodValidationPostProcessor validationPostProcessor() {
		MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
		processor.setAdaptConstraintViolations(true);
		return processor;
	}
}
```

`MethodValidationException` 包含一个 `ParameterValidationResults` 列表，该列表按方法参数分组错误，每个参数都暴露一个 `MethodParameter` 、参数值以及一个 `MessageSourceResolvable` 错误列表（该列表由 `ConstraintViolations` 转换而来）。对于带有字段和属性级联违规的 `@Valid` 方法参数， `ParameterValidationResult` 实例为 `ParameterErrors` （该类实现 `org.springframework.validation.Errors` 接口），并以 `FieldErrors` 形式暴露验证错误。

### 自定义验证错误
经过适配的 `MessageSourceResolvable` 错误可通过配置的 `MessageSource` 转换为错误消息，并基于用户 `Locale` 和语言设置向用户显示。
```
record Person(@Size(min = 1, max = 10) String name) {
}

@Validated
public class MyService {

	void addStudent(@Valid Person person, @Max(2) int degrees) {
		// ...
	}
}
```
如果 `Person.name()` 验证失败， `ConstraintViolation` 会被适配为 `FieldError` 对象：
+ Error codes ： "Size.person.name", "Size.name", "Size.java.lang.String", and "Size"
+ Message arguments "name", 10, and 1 (the field name and the constraint attributes)
+ Default message "size must be between 1 and 10"

要自定义默认消息，可使用上述任意错误代码和消息参数向 `MessageSource` 资源包添加属性。请注意，消息参数 "name" 本身即为 `MessageSourceResolvable` ，包含错误代码 `person.name` 和 `name` ，同样可进行自定义。例如：
```
Size.person.name=Please, provide a {0} that is between {2} and {1} characters long
person.name=username
```

如果 `degrees` 验证失败，会被适配为 `MessageSourceResolvable` 包含以下
+ Error codes "Max.myService#addStudent.degrees", "Max.degrees", "Max.int", "Max"
+ Message arguments "degrees" and 2 (the field name and the constraint attribute)
+ Default message "must be less than or equal to 2"

可以使用以下资源来定义错误消息：
```
Max.degrees=You cannot provide more than {1} {0}
```

### 更多配置
默认的 `LocalValidatorFactoryBean` 配置已能满足大多数情况。针对各种 Bean Validation 构造，存在多种配置选项，涵盖从消息插值到遍历解析等多个方面。有关这些选项的更多信息，请参阅 `LocalValidatorFactoryBean` 的 javadoc 文档。


## 配置 DataBinder
可以使用验证器配置 `DataBinder` 实例。配置完成后，可通过调用 `binder.validate()` 方法调用 `Validator` 。任何验证错误都会自动添加到绑定器的 `BindingResult` 中。
```
Foo target = new Foo();
DataBinder binder = new DataBinder(target);
binder.setValidator(new FooValidator());

// bind to the target object
binder.bind(propertyValues);

// validate the target object
binder.validate();

// get BindingResult that includes any validation errors
BindingResult results = binder.getBindingResult();
```

还可以通过 `dataBinder.addValidators` 和 `dataBinder.replaceValidators` 方法为 `DataBinder` 配置多个 `Validator` 实例。当需要将全局配置的 Bean 验证与在 `DataBinder` 实例上本地配置的 Spring `Validator` 结合使用时，此方法非常有用。