# Spring Mvc Controller Advice
{docsify-updated}

`@ExceptionHandler` 、 `@InitBinder` 和 `@ModelAttribute` 方法只会在声明它们的 `@Controller` 类或类层次结构中起作用。相反，如果在 `@ControllerAdvice` 或 `@RestControllerAdvice` 类中声明了这些方法，那么它们将适用于任何 Controller 或者指定 Controller 。此外，自 5.3 起，`@ControllerAdvice` 中的 `@ExceptionHandler` 方法可用于处理来自任何 `@Controller` 或任何其他处理程序的异常。

`@ControllerAdvice` 通过 `@Component` 进行元标注，因此可以通过组件扫描注册为 Spring Bean。 `@RestControllerAdvice` 通过 `@ControllerAdvice` 和 `@ResponseBody` 进行元标注，这意味着其中的 `@ExceptionHandler` 方法的返回值将通过响应体消息转换呈现，而不是通过 HTML 视图呈现。

启动时， `RequestMappingHandlerMapping` 和 `ExceptionHandlerExceptionResolver` 会检测 controller advice bean 并在程序运行时动地态应用它们。 `@ControllerAdvice` 中的 `@ExceptionHandler` 方法会在 `@Controller` 中处理请求的方法之后运行。相比之下，而 `@ModelAttribute` 和 `@InitBinder` 方法会在请求处理方法运行之前运行。

`@ControllerAdvice` 注解具有一些属性，可以缩小适用的控制器和处理程序的范围。例如
```
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```
选择器在运行时进行评估，如果大量使用，可能会对性能产生负面影响。

## 原理
特定类型的继承关系 `HandlerMethodReturnValueHandler` -> `AbstractMessageConverterMethodProcessor` -> `HttpEntityMethodProcessor/RequestResponseBodyMethodProcessor` 会调用 Advice ，核心代码在: 
```
AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters(...){
    ...
    HttpInputMessage msgToUse =
	    getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
    ...
}
```
<cennter><img src="pics/readeAdvice.png" alt=""></cennter>


```
AbstractMessageConverterMethodArgumentResolver.writeWithMessageConverters(...){
   body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
}
```
<cennter><img src="pics/writeAdvice.png" alt=""></cennter>

最终调用功能都是在 `RequestResponseBodyAdviceChain` 类对应的方法里完成的：
```
@Override
public HttpInputMessage beforeBodyRead(HttpInputMessage request, MethodParameter parameter,
        Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

    for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
        if (advice.supports(parameter, targetType, converterType)) {
            request = advice.beforeBodyRead(request, parameter, targetType, converterType);
        }
    }
    return request;
}

@Override
@Nullable
public Object beforeBodyWrite(@Nullable Object body, MethodParameter returnType, MediaType contentType,
        Class<? extends HttpMessageConverter<?>> converterType,
        ServerHttpRequest request, ServerHttpResponse response) {

    return processBody(body, returnType, contentType, converterType, request, response);
}

@Nullable
private <T> Object processBody(@Nullable Object body, MethodParameter returnType, MediaType contentType,
        Class<? extends HttpMessageConverter<?>> converterType,
        ServerHttpRequest request, ServerHttpResponse response) {

    for (ResponseBodyAdvice<?> advice : getMatchingAdvice(returnType, ResponseBodyAdvice.class)) {
        if (advice.supports(returnType, converterType)) {
            body = ((ResponseBodyAdvice<T>) advice).beforeBodyWrite((T) body, returnType,
                    contentType, converterType, request, response);
        }
    }
    return body;
}
```

从以上代码可以看出，首先会根据请求值/返回值**动态获取**匹配的 `RequestBodyAdvice` 和 `ResponseBodyAdvice` ，然后调用 `supports` 方法判断是否支持当前请求/返回值，如果支持则调用 `beforeBodyRead` 和 `beforeBodyWrite` 方法。

这部分是动态获取的，所以官方文档才会说可能会有运行时开销。


## 接口加解密功能
使用 `RequestBodyAdvice` 和 `ResponseBodyAdvice` 实现基于注解的接口加解密功能。

### 自定义注解
自定义一个注解，只有请求处理的方法参数中包含了注解才会启动加解密功能：
```
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Encrypted {
    String assignee() default "";
}
```

### 接口解密处理
```
@Slf4j
@RestControllerAdvice(assignableTypes = {LoginController.class})
@AllArgsConstructor
public class DecryptRequestAdvice implements RequestBodyAdvice {

    private final AppIdService appIdService;

    @Override
    public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return ObjectUtils.allNotNull(methodParameter.getMethodAnnotation(Encrypted.class));
    }

    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {
        String encryptedBody = new BufferedReader(new InputStreamReader(inputMessage.getBody()))
                .lines().collect(Collectors.joining(System.lineSeparator()));

        try {
            Encrypted methodAnnotation = parameter.getMethodAnnotation(Encrypted.class);
            String assignee = methodAnnotation.assignee();
            String decrypted = AesUtil.decrypt(encryptedBody, getKey(assignee));
            InputStream decryptedInputStream = new ByteArrayInputStream(decrypted.getBytes(StandardCharsets.UTF_8));
            return new HttpInputMessage() {
                @Override
                public InputStream getBody() {
                    return decryptedInputStream;
                }

                @Override
                public org.springframework.http.HttpHeaders getHeaders() {
                    return inputMessage.getHeaders();
                }
            };
        } catch (Exception e) {
            log.info("请求解密失败");
            throw new RuntimeException("请求解密失败", e);
        }
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return body;
    }

    @Override
    public Object handleEmptyBody(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return body;
    }


    private String getKey(String assignee) {
        return appIdService.getSecretByAssignee(assignee);
    }
}
```

### 接口加密处理
```
@RestControllerAdvice(assignableTypes = {LoginController.class})
@AllArgsConstructor
public class EncryptResponseAdvice implements ResponseBodyAdvice<Object> {

    private final AppIdService appIdService;

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return ObjectUtils.allNotNull(returnType.getMethodAnnotation(Encrypted.class));
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        try {
            String responseJson = JacksonUtil.writeValueAsString(body);
            Encrypted methodAnnotation = returnType.getMethodAnnotation(Encrypted.class);
            String assignee = methodAnnotation.assignee();
            return AesUtil.encrypt(responseJson, appIdService.getSecretByAssignee(assignee));
        } catch (Exception e) {
            throw new RuntimeException("响应加密失败", e);
        }
    }
}
```

### 接口
这里注意：  
响应类型需要定义成 `ResponseEntity<String>` 而不能是一个自定义 DTO，应为如果是 DTO，spring 还会用类似 `MappingJackson2HttpMessageConverter` 的转换器处理最后的加密字符串，这样客户端收到的响应会比加密字符串前后加上双引号。
```
@PostMapping("/test")
@Encrypted(assignee = "va")
public ResponseEntity<String> test(@RequestBody TestRequest request) {
    return ResponseEntity.ok(JacksonUtil.writeValueAsString(testService.test(request)));
}
```
