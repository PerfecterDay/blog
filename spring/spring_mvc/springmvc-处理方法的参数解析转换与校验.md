#  SpringMvc 的请求处理方法请求参数绑定与校验
{docsify-updated}

- [SpringMvc 的请求处理方法请求参数绑定与校验](#springmvc-的请求处理方法请求参数绑定与校验)
    - [本文解决的核心问题](#本文解决的核心问题)
    - [参数绑定- HandlerMethodArgumentResolver 及 HttpMessageConverter](#参数绑定--handlermethodargumentresolver-及-httpmessageconverter)
    - [HandlerMethodArgumentResolver 和 HttpMessageConverter 的关系](#handlermethodargumentresolver-和-httpmessageconverter-的关系)
    - [常见的 HandlerMethodArgumentResolver](#常见的-handlermethodargumentresolver)
    - [数据绑定流程剖析](#数据绑定流程剖析)
  - [请求参数验证](#请求参数验证)
    - [Spring 参数校验的原理](#spring-参数校验的原理)
    - [JSR-303](#jsr-303)
    - [Spring 使用  validation 的步骤](#spring-使用--validation-的步骤)
    - [常见的校验注解](#常见的校验注解)
    - [自定义校验注解](#自定义校验注解)
    - [@InitBinder](#initbinder)
  - [Spring MVC中的数据校验](#spring-mvc中的数据校验)

<center><img src="pics/spring-http-journey.webp"></center>


### 本文解决的核心问题
**一个Http请求中的参数或者Json请求体是如何被转换成Controller中的参数对象的 ？**

直接看一个例子，添加自定义的 `HandlerMethodArgumentResolver` 和 `HttpMessageConverter`：
```
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.clear();
        resolvers.add(new HandlerMethodArgumentResolver() {
            @Override
            public boolean supportsParameter(MethodParameter parameter) {
                return true;
            }

            @Override
            public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
                System.out.println("abc");
                return null;
            }
        });
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.clear();
        converters.add(new HttpMessageConverter<Object>() {
            @Override
            public boolean canRead(Class<?> clazz, MediaType mediaType) {
                return true;
            }

            @Override
            public boolean canWrite(Class<?> clazz, MediaType mediaType) {
                return true;
            }

            @Override
            public List<MediaType> getSupportedMediaTypes() {
                List<MediaType> mediaTypes = new ArrayList<>();
                mediaTypes.add(MediaType.ALL);
                return mediaTypes;
            }

            @Override
            public Object read(Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
                System.out.println("AAA");
                return null;
            }

            @Override
            public void write(Object o, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
                System.out.println("BBB");

            }
        });
    }
}

@GetMapping("/test")
public Response test(User user){
	return Response.buildSuccess();
}
```

### 参数绑定- HandlerMethodArgumentResolver 及 HttpMessageConverter
Spring处理参数绑定时首先会根据参数信息寻找合适的 `HandlerMethodArgumentResolver`(比如参数用 `@RequestBody` 注解修饰，就会用内置 `RequestResponseBodyMethodProcessor` 的解析器)，上例中没有用 `@RequestBody` 注解，就会使用我们自定义的参数解析器解析参数。通常在解析器内部会根据 MediaType 不同使用不同 `HttpMessageConverter` 转换 Http 请求。

Spring 会根据请求方法签名的不同，将请求消息中的信息 以一定的方式转换并绑定到请求方法的入参中。当请求消息到达真正需要调用的方法时(如指定的业务方法)，Spring MVC 还有很多工作要做，包括数据转换、数据格式化及数据校验等。

常见的参数解析器： `PathVariableMethodArgumentResolver` , `RequestResponseBodyMethodProcessor`, `RequestHeaderMethodArgumentResolver`，
会用到很多 `HttpMessageConverter`

```
RequestMappingHandlerAdapter.invokeHandlerMethod()->ServletInvocableHandlerMethod.invokeAndHandle()
->InvocableHandlerMethod.invokeForRequest()->HandlerMethodArgumentResolverComposite.resolveArgument()
->RequestResponseBodyMethodProcessor.resolveArgument() -> HttpMessageConverter.read()
```

`HandlerMethodArgumentResolver` 的调用过程:
```
HandlerMethodArgumentResolver（RequestResponseBodyMethodProcessor） -> HttpMessageConverter（MappingJackson2HttpMessageConverter）
-> WebDataBinder 
```

`RequestResponseBodyMethodProcessor` 的核心代码：
```
// RequestResponseBodyMethodProcessor 的 resolveArgument 方法
@Override
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
	parameter = parameter.nestedIfOptional();
	Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType()); //调用 HttpMessageConverter 方法
	String name = Conventions.getVariableNameForParameter(parameter);

	if (binderFactory != null) {
		WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
		if (arg != null) {
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
				throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
			}
		}
		if (mavContainer != null) {
			mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
		}
	}
	return adaptArgumentIfNecessary(arg, parameter);
}
```

### HandlerMethodArgumentResolver 和 HttpMessageConverter 的关系

`HandlerMethodArgumentResolver` 的职责是将 http 请求映射到处理器的方法参数，并且会处理一些参数验证逻辑，这点可以从其定义的方法看出来：
```
public interface HandlerMethodArgumentResolver {
    boolean supportsParameter(MethodParameter parameter);

    @Nullable
    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}
```
在解析参数绑定的过程中会用到 `HttpMessageConverter` 的方法。而 `HttpMessageConverter` 主要是将 http消息转换成一个特定对象以及将一个对象转换成Http消息：
```
public interface HttpMessageConverter<T> {
    boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

    boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

    List<MediaType> getSupportedMediaTypes();

    default List<MediaType> getSupportedMediaTypes(Class<?> clazz) {
        return !this.canRead(clazz, (MediaType)null) && !this.canWrite(clazz, (MediaType)null) ? Collections.emptyList() : this.getSupportedMediaTypes();
    }

    T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;

    void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException;
}
```

### 常见的 HandlerMethodArgumentResolver
+ `RequestResponseBodyMethodProcessor` ： 用 `@RequestBody` 注解的参数会用这个解析
+ `RequestHeaderMethodArgumentResolver` ： 用 `@RequestHeader("xxxx") String var` 注解的参数会用这个解析
+ `RequestParamMethodArgumentResolver` ： 用 `@RequestParam("xxxx") String var` 注解的参数会用这个解析
+ `PathVariableMethodArgumentResolver` : 用 `@PathVariable("xxx") String var` 注解的参数会用这个解析
+ `PathVariableMapMethodArgumentResolver` : 用 `@PathVariable Map map` 注解且没有指定路径参数名的 Map 类型参数会用这个解析
+ `MatrixVariableMethodArgumentResolver` : 用 `@MatrixVariable("xxx") String var` 注解且没有指定矩阵参数名的参数会用这个解析
+ `MatrixVariableMapMethodArgumentResolver` : 用 `@MatrixVariable Map map` 注解且没有指定矩阵参数名的 Map 类型参数会用这个解析
+ `ServletModelAttributeMethodProcessor`：用 `@ModelAttribute` 注解或者没有注解的自定义类型的参数会用这个解析


### 数据绑定流程剖析
Spring MvC通过反射机制对目标处理方法的签名进行分析，将请求消息绑定到处理方法的入参中。数据鄉定的核心部件是 `DataBinder` ，其运行机制描述如下所示：
<center><img src="pics/databind.jpg" width="40%"></center>

Spring MVC 主框架将 ServletRequest 对象及处理方法的入参对象实例传递给 DataBinder, DataBinder 首先调用装配在SpringWeb 上下文中的 ConversionService 组件进行数据类型转换、数据格式化等工作，将 ServletRequest 中的消息填充到入参对象中， 然后调用 Validator 组件对己经鄉定了请求消息数据的入参对象进行数据合法性校验，最终生成数据绑定结果 BindingResult 对象。 BindingResult 包含了已完成数据绑定的入参 对象，还包含相应的校验错误对象。Spring MVC 抽取 BindingResult 中的入参对象及校验错误对象，将它们赋给处理方法的相应入参。


https://www.baeldung.com/spring-data-redis-pub-sub

https://github.com/Homebrew/discussions/discussions/2530
https://www.baeldung.com/spring-retry

## 请求参数验证

### Spring 参数校验的原理



### JSR-303
JSR-303是Java为Bean数据合法性校验提供的标准框架，它定义了一套可标注在成员变量，属性方法上的校验注解。
Hibernate Validation提供了这套标准的实现，在我们引入Spring boot starter validation的时候，默认会引入Hibernate Validation。

### Spring 使用  validation 的步骤
1. 为业务对象bean 添加相应的验证注解  
	```
	@Data
	public class User {
	// 名字不允许为空，并且名字的长度在2位到30位之间
	// 如果名字的长度校验不通过，那么提示错误信息
	@NotNull
	@Size(min=2, max=30,message = "请检查名字的长度是否有问题")
	private String name;

	// 不允许为空，并且年龄的最小值为18
	@NotNull
	@Min(18)
	private Integer age;
	}
	```
2. 控制器内对验证对象前加上 @Valid 注解，验证结果会返回到 BindingResult 对象中  
	```
	// 1. 要校验的参数前，加上@Valid注解
	// 2. 紧随其后的，跟上一个BindingResult来存储校验信息
	@RequestMapping("/test1")
	public Object test1(@Valid User user,BindingResult bindingResult) {
	//如果检验出了问题，就返回错误信息
	// 这里我们返回的是全部的错误信息，实际中可根据bindingResult的方法根据需要返回自定义的信息。
	// 通常的解决方案为：JSR-303 + 全局ExceptionHandler
	if (bindingResult.hasErrors()){
	return bindingResult.getAllErrors();
	}
	return "OK";
	} 
	```

### 常见的校验注解
JSR-303 提供的标准注解：
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

2、编写具体的实现类

我们知道注解只是一个标记，真正的逻辑还要在特定的类中实现，上一步的注解指定了实现校验功能的类为 `IsMobileValidator` 。
```
// 自定义注解一定要实现ConstraintValidator接口奥，里面的两个参数
// 第一个为 具体要校验的注解
// 第二个为 校验的参数类型
public class IsMobileValidator implements ConstraintValidator<IsMobile, String> {

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

### @InitBinder
`@Controller` 或 `@ControllerAdvice` 类可以使用 `@InitBinder` 方法来初始化 `WebDataBinder` 实例，而 `WebDataBinder` 实例反过来也可以使用 `@InitBinder` 方法来初始化 `WebDataBinder` 实例：

+ 将请求参数（即表单或查询数据）绑定到模型对象。
+ 将基于字符串的请求值（如请求参数、路径变量、标题、cookie 等）转换为控制器方法参数的目标类型。
+ 在呈现 HTML 表单时，将模型对象值格式化为字符串值。

`@InitBinder` 方法可以注册特定于控制器的 `java.beans.PropertyEditor` 或 Spring `Converter` 和 `Formatter` 组件。此外，您还可以使用 MVC 配置在全局共享的 `FormattingConversionService` 中注册转换器和格式器类型。
```
@Controller
public class FormController {

	@InitBinder
	public void initBinder(WebDataBinder binder) {
		SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
		dateFormat.setLenient(false);
		binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
		binder.setAllowedFields("oldEmailAddress", "newEmailAddress");
	}
}
```

## Spring MVC中的数据校验
默认情况下，如果类路径上存在 Bean Validation（例如 Hibernate Validator），则 `LocalValidatorFactoryBean` 会注册为全局 `Validator` ，以便在控制器方法参数上与 `@Valid` 和 `@Validated` 配合使用。
也可以自定义注入 `Validator` ：
```
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

	@Override
	public Validator getValidator() {
		// ...
	}
}
```

如果只想要在一个 controller 中使用某个特定的 `Validator` ，而不是全局注册，则可以使用如下：
```
@Controller
public class MyController {

	@InitBinder
	protected void initBinder(WebDataBinder binder) {
		binder.addValidators(new FooValidator());
	}
}
```

`@InitBinder` 是Spring MVC中的一个注解，用于在控制器中自定义数据绑定初始化的方法。它的作用主要是在处理请求之前，对请求中的数据进行初始化绑定，可以用于以下几个方面：
1. 数据预处理：通过@InitBinder注解的方法可以对表单提交的数据进行预处理，比如将日期字符串转换为Date对象、将字符串中的空格去除等。这可以确保请求数据在进入控制器方法之前已经被正确处理和转换，避免了在业务方法中重复处理数据的逻辑。
2. 数据校验：@InitBinder可以用于注册校验器（Validator），对请求中的数据进行校验。通过在@InitBinder方法中注册校验器，可以在数据绑定时对请求参数进行校验，确保数据的有效性。
3. 自定义数据绑定：有时候，Spring MVC默认的数据绑定规则可能无法满足业务需求，需要自定义数据绑定规则。通过在@InitBinder方法中注册自定义的属性编辑器（PropertyEditor），可以实现对特定类型数据的自定义绑定，比如将字符串转换为自定义对象。
4. 全局数据绑定配置：@InitBinder方法可以在控制器内部或者全局配置中使用。如果在控制器内部使用，它只对该控制器的处理方法有效；如果在全局配置中使用，则对所有的控制器都有效。