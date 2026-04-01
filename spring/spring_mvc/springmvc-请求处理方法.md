# Spring-MVC 中的请求处理方法
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods.html

## 方法参数
使用Springmvc 提供的注解可以直接为请求处理方法获取一些请求中的相应内容。

### 直接使用的参数
SpringMvc 会将请求进行处理并包装成对应的对象，这些对象可以直接作为映射方法的实参注入，映射方法可以使用这些对象获取请求相关的信息进行处理。
```
WebRequest, NativeWebRequest
jakarta.servlet.ServletRequest, jakarta.servlet.ServletResponse
java.io.InputStream, java.io.Reader
java.io.OutputStream, java.io.Writer
HttpEntity
jakarta.servlet.http.HttpSession
jakarta.servlet.http.PushBuilder
java.security.Principal
HttpMethod
java.util.Locale
java.util.TimeZone + java.time.ZoneId
java.util.Map, org.springframework.ui.Model, org.springframework.ui.ModelMap
RedirectAttributes
Errors, BindingResult
UriComponentsBuilder
```
如果方法参数类型不属于上述类型，并且是简单类型（由 `BeanUtils#isSimpleProperty` 决定），则将其看作是请求参数，用 `@RequestParam` 解析。否则，将作为 `@ModelAttribute` 的模型参数。

#### HttpEntity
`HttpEntity` 与使用 `@RequestBody` 大致相同，但它基于一个容器对象，该容器对象暴露了请求头和请求体:
```
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
	// ...
}
```

### 使用注解获取相应参数值
SpringMvc 还提供了很多注解用来获取请求中的内容。

#### 获取矩阵变量 @PathVariable和@MatrixVariable
[RFC 3986](https://tools.ietf.org/html/rfc3986#section-3.3) 讨论了路径段中的名-值对。在Spring MVC中，将其称为 "矩阵变量"，但它们也可以被称为URI路径参数。  

矩阵变量可以出现在任意路径段中，每个变量之间用分号隔开，多个值之间用逗号隔开（例如，/cars;color=red,green;year=2012）。也可以通过重复变量名指定多个值（例如，color=red;color=green;color=blue）。

如果预期 URL 包含矩阵变量，控制器方法的请求映射必须使用 URI 变量来屏蔽变量内容，并确保请求能够成功匹配，而与矩阵变量的顺序和存在无关。下面的示例使用了矩阵变量：
```
// GET /owners/42;q=11/pets/21;q=22/2
@GetMapping("/owners/{ownerId}/pets/{petId}/{id}")
public void findPet(
		@MatrixVariable(name="q", pathVar="ownerId") int q1,
		@MatrixVariable(name="q", pathVar="petId") int q2,
		@PathVariable String id) {
	// q1 == 11
	// q2 == 22
	// id == 2
}
```

 使用语法为 `{varName:regex}` 的正则表达式声明 URI 变量。例如，在给定 URL"/spring-web-3.0.5.jar"的情况下，下面的方法会提取文件名、版本和文件扩展名：
```
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
	// ...
}
```

URI 路径模式还可以嵌入 `${...}` 占位符，这些占位符在启动时通过使用 `PropertySourcesPlaceholderConfigurer` 从配置文件中注入。例如，您可以使用它根据某些外部配置对基本 URL 进行参数化。

#### 获取请求参数 @RequestParam
您可以使用 `@RequestParam` 注解将 Servlet 请求参数（即查询参数或表单数据）绑定到控制器中的方法参数。
```
@GetMapping
public String setupForm(@RequestParam("petId") int petId, Model model) { 
	Pet pet = this.clinic.loadPet(petId);
	model.addAttribute("pet", pet);
	return "petForm";
}
```

#### 获取请求头参数 @RequestHeader
`@RequestHeader` 可以获取请求头中的参数。
```
@GetMapping("/demo")
public void handle(
		@RequestHeader("Accept-Encoding") String encoding,
		@RequestHeader("Keep-Alive") long keepAlive) {
	//...
}
```
当 `@RequestHeader` 注解被用于 `Map<String,String>、MultiValueMap<String,String> 或 HttpHeaders` 参数时，该 `Map` 将被填充所有头值。


#### 获取Cookie值 @CookieValue
可以使用 `@CookieValue` 注解将 HTTP cookie 的值绑定到控制器中的方法参数。
```
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) {
	//...
}
```

#### @SessionAttributes 存储会话属性
`@SessionAttributes` 用于在请求之间的会话Session中存储模型属性。它是一个类级别的注解，用于声明特定控制器使用的会话属性。这通常列出了模型属性的名称或模型属性的类型，这些属性应透明地存储在会话中，供后续请求访问。  
在第一次请求中，当一个名称为 pet 的模型属性被添加到模型中时，它被自动提升并保存在 HTTP Servlet 会话中。如下例所示，在另一个控制器方法使用 SessionStatus 方法参数清除存储之前，该属性一直保留在会话中：
```
@Controller
@SessionAttributes("pet")
public class EditPetForm {
	@PostMapping("/pets/{id}")
	public String handle(Pet pet, BindingResult errors, SessionStatus status) {
		if (errors.hasErrors) {
			// ...
		}
		status.setComplete();
		// ...
	}
}
```

#### @SessionAttribute 获取会话属性
如果您需要访问可能存在也可能不存在的已有会话属性，您可以在方法参数上使用 `@SessionAttribute` 注解，如下例所示：
```
@RequestMapping("/")
public String handle(@SessionAttribute User user) {
	// ...
}
```

#### @RequestBody 获取请求体
可以使用 `@RequestBody` 注解通过 `HttpMessageConverter` 将请求 body 读取并反序列化为对象。下面的示例使用了 `@RequestBody` 参数：
```
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
	// ...
}
```

## 方法返回值
```
@ResponseBody
HttpEntity<T>, ResponseEntity<T>
HttpHeaders
ErrorResponse
ProblemDetail
String
View
java.util.Map, org.springframework.ui.Model
ModelAndView
void
DeferredResult<V>
Callable<V>
ListenableFuture<V>, java.util.concurrent.CompletionStage<V>, java.util.concurrent.CompletableFuture<V>
ResponseBodyEmitter, SseEmitter
StreamingResponseBody
```
如果返回值不是以上任何情况，它将被视为模型属性。除非它是由 `BeanUtils#isSimpleProperty` 决定的简单类型，在这种情况下，它会被直接返回而不会解析。

### 响应相关的注解

#### @ResponseBody
可以在方法上使用 `@ResponseBody` 注解，通过 `HttpMessageConverter` 将返回序列化为响应体:
```
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
}
```
`@ResponseBody` 在类级别也被支持，在这种情况下，它被所有控制器方法继承。这就是 `@RestController` 的效果，它只不过是一个标有 `@Controller` 和 `@ResponseBody` 的元注解。

#### ResponseEntity
`ResponseEntity` 类似于 `@ResponseBody` ，但带有状态和头信息。
```
@GetMapping("/something")
public ResponseEntity<String> handle() {
	String body = ... ;
	String etag = ... ;
	return ResponseEntity.ok().eTag(etag).body(body);
}
```

Spring MVC支持异步生成ResponseEntity，或为主体使用单值和多值反应类型。这允许以下类型的异步响应：
+ `ResponseEntity<Mono<T>>` 或 `ResponseEntity<Flux<T>>` 可立即知道响应状态和标头，而正文则在稍后异步提供。如果正文由 0...1 个值组成，请使用 Mono；如果可以产生多个值，请使用 Flux。
+ `Mono<ResponseEntity<T>>` 可在稍后异步提供所有三项内容--响应状态、头信息和正文。这允许响应状态和头信息根据异步请求处理的结果而变化。