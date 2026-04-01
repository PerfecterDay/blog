# Spring-MVC 请求映射 HandlerMapping 配置
{docsify-updated}

`HandlerMapping` 就是将一个请求映射到一个处理请求的执行链 `HandlerExecutionChain` 。
```java
public interface HandlerMapping {
	@Nullable
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

SpringMVC 提供了多种方式来配置 `HandlerMapping` 。默认配置的是：
```
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
```

`RequestMappingHandlerMapping` 提供了使用 `@Controller/@RestController` 注解支持配置的功能。

## @Controller和@RestController
将一个类标记为 Controller bean，这样 Spring MVC 的 RequestMapping 就会扫描到这个 bean 。

## 注册 handlerMapping 请求映射

### @RequestMapping
可以使 `@RequestMapping` 注解将请求映射到控制器方法。它有多种属性，可通过URL、HTTP方法、请求参数、头和媒体类型进行匹配。可以在类级别使用它来表达共享映射，也可以在方法级别使用它来缩小特定端点映射的范围。
RequestMapping 有很多属性可以指定匹配的细节：
+ `consumes` ： 匹配`Content-Type`是指定值的请求 ，`!text/plain` 指除 `text/plain` 以外的任何内容类型。
+ `params` : 匹配指定 `parameter=value` 的请求
+ `headers` : 匹配指定 `myHeader=myValue` 的请求,请注意，在 curl 语法中，头关键字和头值之间用冒号隔开，这与 HTTP 规范相同，而在 Spring 中则使用等号。
+ `produces` : 匹配特定的 `Accept` 请求头， 并且处理器会返回特定 `Accept` 指定的的响应媒体类型


可以在类级别使用 `consumes` 属性。在类级别使用时，方法级别的 `consumes` 属性会覆盖。
```
@PostMapping(path = "/pets", consumes = "application/json", ) 
```

`@RequestMapping` 方法可以使用 URL 模式进行映射。有两种选择：
1. `PathPattern` - 与URL路径匹配的预解析模式，也作为PathContainer进行预解析。该解决方案专为网络使用而设计，可有效处理编码和路径参数，并能高效匹配。
2. `AntPathMatcher` - 将字符串模式与字符串路径相匹配。这是最初的解决方案，也用于Spring配置，以选择classpath、文件系统和其他位置上的资源。它的效率较低，而且字符串路径输入对于有效处理编码和URL的其他问题是一个挑战。

**可以在多个 `Controller` 类上使用相同的匹配路径，比如在A\B两个类上都加上 `@RequestMapping("/test")` 注解，这是可以的，但是A/B 类中的处理方法匹配的子路径不能也相同，也就是说一个完整的 URL 只能匹配到一个方法。**


### 特定方法的
`@RequestMapping` 也有 HTTP 方法特定的快捷方式变体：
+ `@GetMapping`
+ `@PostMapping`
+ `@PutMapping`
+ `@DeleteMapping`
+ `@PatchMapping`

示例：
+ `/resources/ima?e.png`： 匹配路径段中的一个字符
+ `/resources/*.png`： 匹配路径段中的零个或多个字符
+ `/resources/**`： 匹配多个路径段
+ `/projects/{project}/versions`： 匹配路径段并将其作为变量捕获
+ `/projects/{project:[a-z]}/versions` ： 使用 regex 匹配并捕获变量

### 利用代码显示的注册

```java
@Configuration
public class MyConfig {
	@Autowired
	public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler)
			throws NoSuchMethodException {
		RequestMappingInfo info = RequestMappingInfo
				.paths("/user/{id}").methods(RequestMethod.GET).build();
		Method method = UserHandler.class.getMethod("getUser", Long.class);
		mapping.registerMapping(info, handler, method);
	}
}
```

## RequestMappingHandlerMapping 工作原理
`RequestMappingHandlerMapping` 继承自 `AbstractHandlerMethodMapping`，后者实现了生命周期接口 `InitializingBean`：
```java
@Override
public void afterPropertiesSet() {
	initHandlerMethods();
}

/**
* Scan beans in the ApplicationContext, detect and register handler methods.
* @see #getCandidateBeanNames()
* @see #processCandidateBean
* @see #handlerMethodsInitialized
*/
protected void initHandlerMethods() {
	for (String beanName : getCandidateBeanNames()) {
		if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
			processCandidateBean(beanName);
		}
	}
	handlerMethodsInitialized(getHandlerMethods());
}

protected void processCandidateBean(String beanName) {
	...
	if (beanType != null && isHandler(beanType)) {
		detectHandlerMethods(beanName);
	}
}

protected void detectHandlerMethods(Object handler) {
	...
	methods.forEach((method, mapping) -> {
		Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
		registerHandlerMethod(handler, invocableMethod, mapping);
	});
}

// 注册映射信息， 简单理解就是注册 url 和 处理方法的对应关系，详细的看 RequestMappingInfo
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
	this.mappingRegistry.register(mapping, handler, method);
}
```
在回调中会去扫描所有的 bean （这是在实例化阶段，元数据解析阶段已经收集到所有 bean 的信息了，所以这时可以扫描所有 bean），然后通过 `isHandler()` 方法判断是否是请求处理器，如果是的话就将其中的处理方法注册到请求映射中去。 `RequestMappingHandlerMapping` 重写了 `isHandler()` 方法：
```java
@Override
protected boolean isHandler(Class<?> beanType) {
	return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
			AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```
如果bean 用了 `@Controller/@RequestMapping` 注解，则将其视为请求处理器。从以上代码中可以看出，Spring MVC 会使用AOP对映射的处理方法进行AOP代理。

### 查询 URL 匹配
```
RequestMappingInfoHandlerMapping 的

@Override
@Nullable
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	request.removeAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
	try {
		return super.getHandlerInternal(request);
	}
	finally {
		ProducesRequestCondition.clearMediaTypesAttribute(request);
	}
}


AbstractHandlerMethodMapping

@Override
@Nullable
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	String lookupPath = initLookupPath(request);
	this.mappingRegistry.acquireReadLock();
	try {
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
	finally {
		this.mappingRegistry.releaseReadLock();
	}
}
```