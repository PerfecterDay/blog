# Spring-REST Clients
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/integration/rest-clients.html

Spring Framework 为调用 REST 端点提供了以下选项：
+ `RestClient` — 同步调用的 fluent API
+ `WebClient` — 非阻塞响应式 fluent API
+ `RestTemplate` — 采用模板方法 API 的同步客户端（现已弃用，推荐使用 `RestClient` ）
+ `HTTP Service Clients` — 基于生成代理的注解接口

## RestClient
`RestClient` 是一个同步 HTTP 客户端，提供 fluent API 用于执行请求。它作为 HTTP 库的抽象层，负责将 HTTP 请求和响应内容转换为更高层次的 Java 对象，并反之亦然。

### 创建 RestClient
`RestClient` 提供了静态创建快捷方法。它还暴露了一个 `builder()` 方法，支持更多选项：
+ 选择要使用的底层HTTP库
+ 配置消息转换器
+ 设置基础URL
+ 设置默认请求头、Cookie、路径变量及API版本
+ 配置 `ApiVersionInserter`
+ 注册拦截器
+ 注册请求初始化器

一旦创建， `RestClient` 可以被多线程安全的使用。
```
RestClient defaultClient = RestClient.create();

RestClient customClient = RestClient.builder()
	.requestFactory(new HttpComponentsClientHttpRequestFactory()) //设置底层使用的 http 库
	.messageConverters(converters -> converters.add(new MyCustomMessageConverter())) //配置自定义转换器
	.baseUrl("https://example.com")
	.defaultUriVariables(Map.of("variable", "foo"))
	.defaultHeader("My-Header", "Foo")
	.defaultCookie("My-Cookie", "Bar")
	.defaultVersion("1.2")
	.apiVersionInserter(ApiVersionInserter.fromHeader("API-Version").build())
	.requestInterceptor(myCustomInterceptor)
	.requestInitializer(myCustomInitializer)
	.build();
```

#### 选择并配置底层的 http 库
为执行HTTP请求， `RestClient` 采用需要依赖HTTP客户端库。这些库通过 `ClientRequestFactory` 接口进行适配，提供多种实现方案：
+ `JdkClientHttpRequestFactory` ：使用Java JDK的 `HttpClient`
+ `HttpComponentsClientHttpRequestFactory` ：适用于Apache HTTP Components 的 `HttpClient`
+ `JettyClientHttpRequestFactory` ：适用于Jetty的 `HttpClient`
+ `ReactorNettyClientRequestFactory` ：用于 Reactor Netty 的 `HttpClient`
+ `SimpleClientHttpRequestFactory` ：作为简单默认实现

若 RestClient 构建时未指定请求工厂，则优先使用类路径中存在的 Apache 或 Jetty `HttpClient` 。若未找到，且加载了 java.net.http 模块，则使用 Java 的 `HttpClient` 。最后才会采用简单默认实现。

如果需要对底层http client 库进行一些配置，如连接超时时间、连接池大小、read-write timeout 等底层设置时，可以构造一个配置好的 client 对象，并初始化到上述对饮的 `ClientRequestFactory` 实现中，如：

##### JDK HttpClient
```
HttpClient httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))   // 建连超时
                .followRedirects(HttpClient.Redirect.NORMAL)
                .version(HttpClient.Version.HTTP_2)
                .build();

RestClient restClient = RestClient.builder()
		.baseUrl("https://671d110e-33d7-449f-b7f7-7387077781df.mock.pstmn.io")
		.requestFactory(new JdkClientHttpRequestFactory(httpClient))
		.build();

ResponseEntity<ApiResponse> result = restClient.post()
		.uri("https://671d110e-33d7-449f-b7f7-7387077781df.mock.pstmn.io/am5/rest/OATH/create-assign-and-encrypt")
		.contentType(MediaType.APPLICATION_JSON)
		.header("x-api-key", "xxxx")
		.header("x-mock-response-name", "aaa")
		.body("{}")
		.retrieve()
		.toEntity(ApiResponse.class);

System.out.println("Response status: " + result.getStatusCode());
System.out.println("Response headers: " + result.getHeaders());
System.out.println("Contents: " + result.getBody());
```

当你不关心返回值body时，可以使用 `toBodilessEntity()` 方法。

##### Apache HTTP Components
```
PoolingHttpClientConnectionManager cm =
        PoolingHttpClientConnectionManagerBuilder.create()
                .setMaxConnTotal(200)
                .setMaxConnPerRoute(50)
                .build();

CloseableHttpClient client = HttpClients.custom()
        .setConnectionManager(cm)
        .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(3, TimeUnit.SECONDS)
                .setResponseTimeout(5, TimeUnit.SECONDS)
                .build())
        .build();

RestClient restClient = RestClient.builder()
        .requestFactory(new HttpComponentsClientHttpRequestFactory(client))
        .build();
```


#### 异常处理
默认情况下，当 `RestClient` 获取到状态码为 `4xx` 或 `5xx` 的响应时，会抛出 `RestClientException` 的子类异常，比如 `HttpClientErrorException` 。

如果不想抛出异常，可以在每次请求时处理错误码：
```
String result = restClient.get() 
	.uri("https://example.com/this-url-does-not-exist") 
	.retrieve()
	.onStatus(HttpStatusCode::is4xxClientError, (request, response) -> { 
		// throw new MyCustomRuntimeException(response.getStatusCode(), response.getHeaders()); 
		// return null; // will not throw exception
	})
	.body(String.class);
```

或者在创建时设置默认的处理方法：
```
RestClient restClient = RestClient.builder()
	.baseUrl(baseUrl)
	.defaultStatusHandler(
			HttpStatusCode::is4xxClientError,
			(request, response) -> {
				String body = new String(response.getBody().readAllBytes());
				log.info("body is {}", body);
			}
	)
	.build();
```


#### Exchange 方法
对于更高级的场景， `RestClient` 通过 `exchange()` 方法提供对底层 HTTP 请求和响应的访问，该方法可替代 `retrieve()` 使用。使用 `exchange()` 时不能使用 `onStatus()` 的错误处理方法 ，因为 `exchange` 函数本身已提供对完整响应的访问权限，允许您执行任何必要的错误处理。
```
Pet result = restClient.get()
	.uri("https://petclinic.example.com/pets/{id}", id)
	.accept(APPLICATION_JSON)
	.exchange((request, response) -> {
		if (response.getStatusCode().is4xxClientError()) {
			throw new MyCustomRuntimeException(response.getStatusCode(), response.getHeaders());
		}
		else {
			Pet pet = convertResponse(response);
			return pet;
		}
	});
```

#### multipart
```
MultiValueMap<String, Object> parts = new LinkedMultiValueMap<>();

parts.add("fieldPart", "fieldValue");
parts.add("filePart", new FileSystemResource("...logo.png"));
parts.add("jsonPart", new Person("Jason"));

HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_XML);
parts.add("xmlPart", new HttpEntity<>(myBean, headers));

// send using RestClient.post or RestTemplate.postForEntity
```

### Interceptor
下边这个示例展示了如何在请求发送前，对请求进行操作，以及如何操作响应对象：
```
@Bean
IUserCenterClient userCenterClient(@Value("${uc.url}") String baseUrl) {
	RestClient restClient = RestClient.builder()
			.baseUrl(baseUrl)
			.requestInterceptor((request, body, execution) -> {
				request.getHeaders().add("short-token", getCurrentRequestToken());
				ClientHttpResponse response = execution.execute(request, body);
				response.getHeaders().add("foo","bar");
				return response;
			})
			.build();

	HttpServiceProxyFactory proxyFactory = HttpServiceProxyFactory
			.builderFor(RestClientAdapter.create(restClient))
			.build();

	return proxyFactory.createClient(IUserCenterClient.class);

}


private String getCurrentRequestToken() {
	ServletRequestAttributes attrs =
			(ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
	if (attrs != null) {
		return attrs.getRequest().getHeader("short-token");
	}
	return null;
}
```

## WebClient
> https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html

`WebClient` 是一个用于执行 HTTP 请求的非阻塞响应式客户端。它在 5.0 版本中引入，为 `RestTemplate` 提供了替代方案，支持同步、异步和流式处理场景。  
`WebClient` 支持以下功能：
+ Non-blocking I/O
+ Reactive Streams back pressure
+ High concurrency with fewer hardware resources
+ Functional-style, fluent API that takes advantage of Java 8 lambdas
+ Synchronous and asynchronous interactions
+ Streaming up to or streaming down from a server

### 创建
与 `RestClient` 类似， `WebClient` 需要一个 HTTP 客户端库来执行请求。支持以下类型的客户端库：
+ Reactor netty 客户端
+ JDK HttpClient
+ Jetty Reactive Client
+ Apache HttpComponents
+ Others can be plugged in via `ClientHttpConnector`.

The simplest way to create WebClient is through one of the static factory methods:
+ `WebClient.create()`
+ `WebClient.create(String baseUrl)`

You can also use WebClient.builder() with further options:
+ `uriBuilderFactory` : Customized UriBuilderFactory to use as a base URL.
+ `defaultUriVariables` : default values to use when expanding URI templates.
+ `defaultHeader` : Headers for every request.
+ `defaultCookie` : Cookies for every request.
+ `defaultApiVersion` : API version for every request.
+ `defaultRequest` : Consumer to customize every request.
+ `filter` : Client filter for every request.
+ `exchangeStrategies` : HTTP message reader/writer customizations.
+ `clientConnector` : HTTP client library settings.
+ `apiVersionInserter` : to insert API version values in the request
+ `observationRegistry` : the registry to use for enabling Observability support.
+ `observationConvention` : an optional, custom convention to extract metadata for recorded observations.


#### Reactor Netty
```
HttpClient httpClient = HttpClient.create()
		.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
		.responseTimeout(Duration.ofSeconds(2))
		.doOnConnected(conn -> conn
				.addHandlerLast(new ReadTimeoutHandler(10))
				.addHandlerLast(new WriteTimeoutHandler(10)))
		.secure(sslSpec -> ...);

WebClient webClient = WebClient.builder()
		.clientConnector(new ReactorClientHttpConnector(httpClient))
		.build();
```

#### JDK HttpClient
```
HttpClient httpClient = HttpClient.newBuilder()
	.followRedirects(Redirect.NORMAL)
	.connectTimeout(Duration.ofSeconds(20))
	.build();

ClientHttpConnector connector =
		new JdkClientHttpConnector(httpClient, new DefaultDataBufferFactory());

WebClient webClient = WebClient.builder().clientConnector(connector).build();
```

#### Jetty
```
HttpClient httpClient = new HttpClient();
httpClient.setCookieStore(...);

WebClient webClient = WebClient.builder()
		.clientConnector(new JettyClientHttpConnector(httpClient))
		.build();
```

#### HttpComponents
```
HttpAsyncClientBuilder clientBuilder = HttpAsyncClients.custom();
clientBuilder.setDefaultRequestConfig(...);
CloseableHttpAsyncClient client = clientBuilder.build();

ClientHttpConnector connector = new HttpComponentsClientHttpConnector(client);

WebClient webClient = WebClient.builder().clientConnector(connector).build();
```

## HTTP Service Client-OpenFeign 的替代
> https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-service-client

您可以将HTTP服务定义为包含 `@HttpExchange` 方法的Java接口，并使用 `HttpServiceProxyFactory` 基于该接口创建客户端代理，通过 `RestClient` `、WebClient` 或 `RestTemplate` 实现HTTP远程访问。在服务器端， `@Controller` 类可实现相同接口，通过 `@HttpExchange` 控制器方法处理请求。

```
@HttpExchange
public interface ISprintClient {
    @PostExchange(url = "/update/user/create")
    ResponseEntity<String> registerUser(@RequestBody UserCreateRequestDTO userCreateRequestDTO);

    @PostExchange(url = "/query/user/findById")
    ResponseEntity<String> queryUser(@RequestBody UserQueryDTO userQueryDTO);

    @PostExchange(url = "/OATH/create-assign-and-encrypt")
    ResponseEntity<String> bindApply(@RequestBody BindRequestDTO bindRequestDTO);
}
```

生成客户端代理配置：
```
@Configuration
public class SprintServiceConfiguration {
    @Bean
    public ISprintClient iSprintService(@Value("${isprint.url}") String baseUrl) {
        RestClient restClient = RestClient.builder()
                .baseUrl(baseUrl)
                .defaultStatusHandler(
                        HttpStatusCode::is4xxClientError,
                        (request, response) -> {
                            String body = new String(response.getBody().readAllBytes());
                            log.info("body is {}", body);
                        }
                )
                .build();
        HttpServiceProxyFactory factory =
                HttpServiceProxyFactory
                        .builderFor(RestClientAdapter.create(restClient))
                        .build();

        return factory.createClient(ISprintClient.class);
    }
}
```

类似的，也可以使用 `WebClient` 或者 `RestTemplate`:
```
// Using RestClient...
RestClient restClient = RestClient.create("...");
RestClientAdapter adapter = RestClientAdapter.create(restClient);

// or WebClient...
WebClient webClient = WebClient.create("...");
WebClientAdapter adapter = WebClientAdapter.create(webClient);

// or RestTemplate...
RestTemplate restTemplate = new RestTemplate();
RestTemplateAdapter adapter = RestTemplateAdapter.create(restTemplate);

HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

RepositoryService service = factory.createClient(RepositoryService.class);

service.getRepository(...)
service.updateRepository(...)
```

服务端实现：
```
@RestController
public class RepoController implements RepositoryService{
    
	@Override
	public Repository getRepository(@PathVariable String owner, @PathVariable String repo){
		...
	}

    @Override
    void updateRepository(@PathVariable String owner, @PathVariable String repo,
			@RequestParam String name, @RequestParam String description, @RequestParam String homepage){
	}
}
```
从 Spring 6 开始：
+ `@HttpExchange` ---> `@RequestMapping`
+ `@GetExchange` ----> `@GetMapping`
+ `@PostExchange` -----> `@PostMapping`
+ `@PatchExchange` 
等注解能被 `HttpServiceProxyFactory` (客户端) 和 `RequestMappingHandlerMapping` (服务端) 同时支持了。同一套注解，既能“发请求”，也能“接请求”。

但是请注意：
**以上做法只适合在内部系统之间调用的场景下适用，不适合于对外 API 的公共接口**。如果对外接口也使用这种方式，会有以下弊端：
+ Controller 层容易被“客户端语义”污染
+ Header / HTTP 细节可能分歧
+ 版本演进困难

HTTP service client 是通过HTTP实现远程访问的强大且富有表现力的选择。它允许团队自主掌握 REST API 的工作原理、哪些部分与客户端应用相关、需要创建哪些输入输出类型、需要哪些端点方法签名、需要哪些Javadoc文档等知识。由此生成的Java API既提供指导又可直接使用。


### 4xx/5xx 错误处理
```
@HttpExchange
public interface ISprintClient {
    @PostExchange(url = "/update/user/create")
    ResponseEntity<String> registerUser(@RequestBody UserCreateRequestDTO userCreateRequestDTO);
}
```
当返回 `4xx/5xx` 错误码时，不会解析响应体并返回到 `ResponseEntity<String>`。这种情况下，使用 `try...catch` 捕获异常，然后从异常中获取相应内容：
```
 try {
	sprintClient.registerUser(userCreateRequestDTO);
	return true;
} catch (HttpClientErrorException e) {
	log.error("创建iSprint用户失败: {}", e);
	String responseBody = e.getResponseBodyAsString();
	Optional<FaultResponseVO> faultResponseVO = Optional.ofNullable(jsonMapper.readValue(responseBody, FaultResponseVO.class));
	return faultResponseVO.map(FaultResponseVO::getFault)
			.map(FaultVO::getCode)
			.filter(code -> code == 7006)
			.isPresent();
}
```

### 注解
1. `AppResponse unTrust(@RequestParam(name = "acct") String acct);` ：使用 `@RequestParam` 注解时必须使用 `name=xxx` 属性显式指明参数名，如果不想显式指定 `AppResponse unTrust(@RequestParam String acct);` 则需要 `javac -parameters` 