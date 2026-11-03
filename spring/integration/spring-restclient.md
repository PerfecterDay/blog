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

## 忽略证书/host匹配的检查
Providing an X509TrustManager that does not do certificate verification (as in the answers above) is insufficient to disable hostname verification because the implementation of SSLContext "wraps" the provided X509TrustManager with an X509ExtendedTrustManager if it is not an X509ExtendedTrustManager. The wrapping "Extended" X509ExtendedTrustManager performs hostname verification.

```
X509ExtendedTrustManager -> AbstractTrustManagerWrapper -> X509TrustManagerImpl

TrustManager[] trustAll = new TrustManager[]{
		new X509ExtendedTrustManager() {
			public X509Certificate[] getAcceptedIssuers() {
				return null;
			}

			public void checkClientTrusted(
					final X509Certificate[] a_certificates,
					final String a_auth_type) {
			}

			public void checkServerTrusted(
					final X509Certificate[] a_certificates,
					final String a_auth_type) {
			}

			public void checkServerTrusted(
					final X509Certificate[] a_certificates,
					final String a_auth_type,
					final Socket a_socket) {
			}

			@Override
			public void checkClientTrusted(X509Certificate[] chain, String authType, Socket socket) throws CertificateException {

			}

			public void checkClientTrusted(
					final X509Certificate[] a_certificates,
					final String a_auth_type,
					final SSLEngine a_engine) {
			}

			public void checkServerTrusted(
					final X509Certificate[] a_certificates,
					final String a_auth_type,
					final SSLEngine a_engine) {
			}
		}
};
SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, trustAll, new SecureRandom());

HttpClient httpClient = HttpClient.newBuilder()
		.connectTimeout(Duration.ofSeconds(3))// 建连超时
		.sslContext(sslContext)
		.build();

RestClient restClient = RestClient.builder()
		.requestFactory(new JdkClientHttpRequestFactory(httpClient))
		.configureMessageConverters(builder -> {
			builder.addCustomConverter(new ClientAuthClientConfiguration.TextHtmlJacksonConverter(new JsonMapper()));
		})
		.baseUrl(smsGatewayProperties.getUrl())
		.build();

HttpServiceProxyFactory httpServiceProxyFactory = HttpServiceProxyFactory
		.builderFor(RestClientAdapter.create(restClient)).build();

return httpServiceProxyFactory.createClient(ISmsClient.class);
```

当使用 JDK 内置的 `HttpClient` 时，如果需要忽略 host 的匹配检查，只能在启动命令行添加 `-Djdk.internal.httpclient.disableHostnameVerification=true` 。

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

### 方法参数
+ `URI`
+ `UriBuilderFactory`
+ `HttpMethod`
+ `@RequestHeader`
+ `@PathVariable`
+ `@RequestAttribute`
+ `@RequestBody`
+ `@RequestParam`
+ `@RequestPart`
+ `MultipartFile`
+ `@CookieValue`

除非将 `required` 属性（若参数注解中提供该属性）设置为 `false` ，或者该参数被标记为可选（由 `MethodParameter#isOptional` 确定），否则方法参数不能为 null。

`RestClientAdapter` 为类型为 `StreamingHttpOutputMessage.Body` 的方法参数提供了额外支持，允许通过向 `OutputStream` 写入数据来发送请求正文。

### Custom Arguments
可以配置一个自定义的 `HttpServiceArgumentResolver` 。下面的示例接口使用了一个自定义的 `Search` 方法参数类型：

```
public interface RepositoryService {
	@GetExchange("/repos/search")
	List<Repository> searchRepository(Search search);
}

static class SearchQueryArgumentResolver implements HttpServiceArgumentResolver {
	@Override
	public boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues) {
		if (parameter.getParameterType().equals(Search.class)) {
			Search search = (Search) argument;
			requestValues.addRequestParameter("owner", search.owner());
			requestValues.addRequestParameter("language", search.language());
			requestValues.addRequestParameter("query", search.query());
			return true;
		}
		return false;
	}
}


RestClient restClient = RestClient.builder().baseUrl("https://api.github.com/").build();
RestClientAdapter adapter = RestClientAdapter.create(restClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory
		.builderFor(adapter)
		.customArgumentResolver(new SearchQueryArgumentResolver())
		.build();
RepositoryService repositoryService = factory.createClient(RepositoryService.class);

Search search = Search.create()
		.owner("spring-projects")
		.language("java")
		.query("rest")
		.build();
List<Repository> repositories = repositoryService.searchRepository(search);
```

### 返回值
+ `void`
+ `HttpHeaders`
+ `<T>`
+ `ResponseEntity<Void>`
+ `ResponseEntity<T>`

### 错误处理
```
// For RestClient
RestClient restClient = RestClient.builder()
		.defaultStatusHandler(HttpStatusCode::isError, (request, response) -> ...)
		.build();
RestClientAdapter adapter = RestClientAdapter.create(restClient);

// or for WebClient...
WebClient webClient = WebClient.builder()
		.defaultStatusHandler(HttpStatusCode::isError, resp -> ...)
		.build();
WebClientAdapter adapter = WebClientAdapter.create(webClient);
```

### Decorating the Adapter
`HttpExchangeAdapter` 和 `ReactorHttpExchangeAdapter` 是用于将 HTTP 接口客户端基础设施与调用底层客户端的细节解耦的契约。它们分别提供了针对 `RestClient` 、 `WebClient` 和 `RestTemplate` 的适配器实现。

有时，通过在 `HttpServiceProxyFactory.Builder` 中可配置的装饰器拦截客户端调用可能会很有用。例如，你可以应用内置装饰器来抑制 404 异常，并返回一个包含 `NOT_FOUND` 状态码且正文为空的  `ResponseEntity ：
```
// For RestClient
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(restClientAdapter)
		.exchangeAdapterDecorator(NotFoundRestClientAdapterDecorator::new)
		.build();

// or for WebClient...
HttpServiceProxyFactory proxyFactory = HttpServiceProxyFactory.builderFor(webClientAdapter)
		.exchangeAdapterDecorator(NotFoundWebClientAdapterDecorator::new)
		.build();
```


### 使用注解配置
使用 `HttpServiceProxyFactory` 创建客户端代理非常简单，但若将其声明为 Bean，则会导致配置重复。此外，可能需要配置多个目标主机，从而需要配置多个客户端，甚至需要创建更多的客户端代理 Bean。

为了更轻松地在大规模环境中处理接口客户端，Spring 框架提供了专门的配置支持。它允许应用程序专注于按组识别 HTTP 服务，并为每个组定制客户端，而框架会透明地创建一个客户端代理注册表，并将每个代理声明为 Bean。

HTTP 服务组仅仅是一组共享相同客户端配置以及用于创建代理的 `HttpServiceProxyFactory` 实例的接口集合。通常，这意味着每个主机对应一个组，但若底层客户端需要不同的配置，也可以为同一目标主机设置多个组。

声明 HTTP 服务组的一种方法是在 `@Configuration` 类中使用 `@ImportHttpServices` 注解，如下所示：
```
@Configuration
@ImportHttpServices(group = "echo", types = {EchoServiceA.class, EchoServiceB.class})
@ImportHttpServices(group = "echo2", types = {EchoServiceA.class})
@ImportHttpServices(group = "greeting", basePackageClasses = GreetServiceA.class)
public class ClientConfig {
}
```

或者也可以使用编程的方式：
```
public class MyHttpServiceRegistrar extends AbstractHttpServiceRegistrar {

	@Override
	protected void registerHttpServices(GroupRegistry registry, AnnotationMetadata metadata) {
		registry.forGroup("echo").register(EchoServiceA.class, EchoServiceB.class);
		registry.forGroup("greeting").detectInBasePackages(GreetServiceA.class);
	}
}

@Configuration
@Import(MyHttpServiceRegistrar.class)
public class ClientConfig {
}
```

声明 HTTP 服务组后，请添加一个 `HttpServiceGroupConfigurer` Bean 来为每个组自定义客户端。例如：
```
@Configuration
@ImportHttpServices(group = "echo", types = {EchoServiceA.class, EchoServiceB.class})
@ImportHttpServices(group = "greeting", basePackageClasses = GreetServiceA.class)
public class ClientConfig {

	@Bean
	public RestClientHttpServiceGroupConfigurer groupConfigurer() {
		return groups -> {
			// configure client for group "echo"
			groups.filterByName("echo").forEachClient((group, clientBuilder) -> ...);

			// configure the clients for all groups
			groups.forEachClient((group, clientBuilder) -> ...);

			// configure client and proxy factory for each group
			groups.forEachGroup((group, clientBuilder, factoryBuilder) -> ...);
		};
	}
}
```

然后直接就可以注入使用了：
```
@Service
public class MyService {

	private final EchoServiceA echoServiceA;

	public MyService(EchoServiceA echoServiceA) {
		this.echoServiceA = echoServiceA;
	}

	public void doSomething() {
		echoServiceA.echo();
	}
}
```

但是，如果存在多个同类型的客户端代理（例如，多个组中存在相同的接口），则该类型不存在唯一的 Bean，因此无法仅通过类型进行自动注入。对于此类情况，可以直接使用存储所有代理的 `HttpServiceProxyRegistry` ，并按组获取所需的代理：
```
@RestController
public class EchoController {
	private final EchoService echoService1;
	private final EchoService echoService2;

	public EchoController(HttpServiceProxyRegistry registry) {
		this.echoService1 = registry.getClient("echo1", EchoService.class);
		this.echoService2 = registry.getClient("echo2", EchoService.class);
	}
}
```

### Springboot 中的支持
在 Springboot 中使用 HTTP Services client 与在 Spring 中基本一致，只是在自动配置上有所不同。主要有
+ 使用 `@HttpExchange` 等声明接口
+ 使用 `@ImportHttpServices` 导入接口，也可以用前边说的编程方式
+ 使用springboot 配置配置相关信息
  ```
    spring.http.clients.connect-timeout=2s
	spring.http.clients.read-timeout=1s
	spring.http.clients.redirects=dont-follow

	spring.http.serviceclient.echo.base-url=https://echo.zuplo.io
	spring.http.serviceclient.echo.connect-timeout=2s
	spring.http.serviceclient.echo.read-timeout=2s
  ```
+ 如果需要自定义配置，可以使用 `RestClientHttpServiceGroupConfigurer/WebClientHttpServiceGroupConfigurer` 或者其它的 `HttpServiceGroupConfigurer` 实现。
  ```
	@Configuration(proxyBeanMethods = false)
	public class MyHttpServiceGroupConfiguration {

		@Bean
		RestClientHttpServiceGroupConfigurer myHttpServiceGroupConfigurer() {
			return (groups) -> groups.forEachClient((group, clientBuilder) -> {
				String groupName = group.name();
				clientBuilder.defaultHeader("service-group", groupName);
			});
		}

	}
  ```
注意，使用以上配置时必须引入 `org.springframework.boot:spring-boot-restclient` 依赖， `HttpServiceClientAutoConfiguration` 会自动配置 `RestClient` 相关的属性。

### 注解
1. `AppResponse unTrust(@RequestParam(name = "acct") String acct);` ：使用 `@RequestParam` 注解时必须使用 `name=xxx` 属性显式指明参数名，如果不想显式指定 `AppResponse unTrust(@RequestParam String acct);` 则需要 `javac -parameters` 


## 原理
`HttpServiceProxyFactory` 中的主要方法：
```
public <S> S createClient(Class<S> serviceType) {
	List<HttpServiceMethod> httpServiceMethods =
			MethodIntrospector.selectMethods(serviceType, this::isExchangeMethod).stream()
					.map(method -> createHttpServiceMethod(serviceType, method))
					.toList();

	return getProxy(serviceType, httpServiceMethods);
}

private <S> HttpServiceMethod createHttpServiceMethod(Class<S> serviceType, Method method) {
	Assert.notNull(this.argumentResolvers,
			"No argument resolvers: afterPropertiesSet was not called");

	return new HttpServiceMethod(
			method, serviceType, this.argumentResolvers, this.requestValuesProcessor,
			this.exchangeAdapter, this.embeddedValueResolver);
}

@SuppressWarnings("unchecked")
private <S> S getProxy(Class<S> serviceType, List<HttpServiceMethod> httpServiceMethods) {
	MethodInterceptor interceptor = new HttpServiceMethodInterceptor(httpServiceMethods);
	ProxyFactory factory = new ProxyFactory(serviceType, interceptor);
	for (ProxyFactoryCustomizer customizer : this.proxyFactoryCustomizers) {
		customizer.customize(factory, serviceType);
	}
	return (S) factory.getProxy(serviceType.getClassLoader());
}
```

可以看出就是创建了 `HttpServiceMethod` 的实例，并且用其作为代理增强 `HttpServiceMethodInterceptor` 的初始化。然后，使用 `ProxyFactory` 生成了代理对象。如果配置了 `ProxyFactoryCustomizer` 还会调用其 `customize` 方法。

`HttpServiceMethodInterceptor` 的 `invoke` 方法如下：
```
@Override
public @Nullable Object invoke(MethodInvocation invocation) throws Throwable {
	Method method = invocation.getMethod();
	HttpServiceMethod httpServiceMethod = this.httpServiceMethods.get(method);
	if (httpServiceMethod != null) {
		@Nullable Object[] arguments = KotlinDetector.isSuspendingFunction(method) ?
				resolveCoroutinesArguments(invocation.getArguments()) : invocation.getArguments();
		return httpServiceMethod.invoke(arguments);
	}
	if (method.isDefault()) {
		if (invocation instanceof ReflectiveMethodInvocation reflectiveMethodInvocation) {
			Object proxy = reflectiveMethodInvocation.getProxy();
			return InvocationHandler.invokeDefault(proxy, method, invocation.getArguments());
		}
	}
	throw new IllegalStateException("Unexpected method invocation: " + method);
}
```

可以看出，最后还是委托 `HttpServiceMethod` 的 `invoke` 方法实现功能。
```
final class HttpServiceMethod {

	private static final boolean REACTOR_PRESENT =
			ClassUtils.isPresent("reactor.core.publisher.Mono", HttpServiceMethod.class.getClassLoader());

	private final Method method;
	private final MethodParameter[] parameters;
	private final List<HttpServiceArgumentResolver> argumentResolvers;
	private final HttpRequestValues.Processor requestValuesProcessor;
	private final HttpRequestValuesInitializer requestValuesInitializer;
	private final ResponseFunction responseFunction;

	HttpServiceMethod(
			Method method, Class<?> serviceType, List<HttpServiceArgumentResolver> argumentResolvers,
			HttpRequestValues.Processor valuesProcessor, HttpExchangeAdapter adapter,
			@Nullable StringValueResolver embeddedValueResolver) {

		this.method = method;
		this.parameters = initMethodParameters(method);
		this.argumentResolvers = argumentResolvers;
		this.requestValuesProcessor = valuesProcessor;

		boolean isReactorAdapter = (REACTOR_PRESENT && adapter instanceof ReactorHttpExchangeAdapter);

		this.requestValuesInitializer =
				HttpRequestValuesInitializer.create(
						method, serviceType, embeddedValueResolver,
						(isReactorAdapter ? ReactiveHttpRequestValues::builder : HttpRequestValues::builder));

		this.responseFunction = (isReactorAdapter ?
				ReactorExchangeResponseFunction.create((ReactorHttpExchangeAdapter) adapter, method, serviceType) :
				ExchangeResponseFunction.create(adapter, method, serviceType));
	}

	public @Nullable Object invoke(@Nullable Object[] arguments) {
		HttpRequestValues.Builder requestValues = this.requestValuesInitializer.initializeRequestValuesBuilder();
		applyArguments(requestValues, arguments);
		this.requestValuesProcessor.process(this.method, this.parameters, arguments, requestValues);
		return this.responseFunction.execute(requestValues.build());
	}
}	
```

`HttpExchangeAdapter` 正是我们在配置 `HttpServiceProxyFactory` 时创建的：
```
HttpServiceProxyFactory httpServiceProxyFactory
                = HttpServiceProxyFactory.builderFor(RestClientAdapter.create(restClient))
                .build();
```

常见的 `HttpExchangeAdapter` 有:
+ `RestClientAdapter`
+ `WebClientAdapter`
+ `RestTemplateAdapter`

这里使用了适配器模式， 比如 `RestClientAdapter` 实现了 `HttpExchangeAdapter` 接口，但是 `RestClient` 没有实现任何接口。`RestClientAdapter` 底层就是将 `RestClient` 的功能适配成 `HttpExchangeAdapter` 需要的形式。

`HttpServiceArgumentResolver/HttpRequestValues.Processor` 等能够处理请求请求对象的类也是在 `HttpServiceMethod` 中发挥作用的。

此外，这套体系中还使用了装饰器模式： `HttpExchangeAdapterDecorator` :
```
public class HttpExchangeAdapterDecorator implements HttpExchangeAdapter {

	private final HttpExchangeAdapter delegate;


	public HttpExchangeAdapterDecorator(HttpExchangeAdapter delegate) {
		this.delegate = delegate;
	}


	/**
	 * Return the wrapped delegate {@code HttpExchangeAdapter}.
	 */
	public HttpExchangeAdapter getHttpExchangeAdapter() {
		return this.delegate;
	}


	@Override
	public boolean supportsRequestAttributes() {
		return this.delegate.supportsRequestAttributes();
	}

	@Override
	public void exchange(HttpRequestValues requestValues) {
		this.delegate.exchange(requestValues);
	}

	@Override
	public HttpHeaders exchangeForHeaders(HttpRequestValues requestValues) {
		return this.delegate.exchangeForHeaders(requestValues);
	}

	@Override
	public <T> @Nullable T exchangeForBody(HttpRequestValues requestValues, ParameterizedTypeReference<T> bodyType) {
		return this.delegate.exchangeForBody(requestValues, bodyType);
	}

	@Override
	public ResponseEntity<Void> exchangeForBodilessEntity(HttpRequestValues requestValues) {
		return this.delegate.exchangeForBodilessEntity(requestValues);
	}

	@Override
	public <T> ResponseEntity<T> exchangeForEntity(HttpRequestValues requestValues, ParameterizedTypeReference<T> bodyType) {
		return this.delegate.exchangeForEntity(requestValues, bodyType);
	}

}
```
默认的 `HttpExchangeAdapterDecorator` 装饰器没有增加任何功能，只是调用被装饰的 `HttpExchangeAdapter delegate` 的方法。但是如果需要增加额外功能，可以重写对应的方法实现自定义逻辑，比如常见的实现有：
+ `CircuitBreakerAdapterDecorator` : 使用 `CircuitBreaker` 实现了熔断降级的功能
+ `NotFoundRestClientAdapterDecorator` ： 包装了 404 错误处理
+ `ReactiveCircuitBreakerAdapterDecorator` 
+ `ReactorHttpExchangeAdapterDecorator` 


`@ImportHttpServices` 使用 `@Import(ImportHttpServiceRegistrar.class)` 导入了 `ImportHttpServiceRegistrar` , 该类继承自 `AbstractHttpServiceRegistrar` , 跟踪它的 `registerBeanDefinitions` 方法。


在 `pring-cloud-circuitBreaker` 中会沿着以下路径增加 `CircuitBreakerAdapterDecorator` 装饰器：
`CommonsClientAutoConfiguration` ---> `CircuitBreakerRestClientHttpServiceGroupConfigurer`  ---> `CircuitBreakerAdapterDecorator` 