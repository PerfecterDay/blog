#  spring-cloud-openfeign
{docsify-updated}
> https://www.baeldung.com/spring-cloud-openfeign  
> https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/

## 简介
Spring Cloud OpenFeign 是一个用于Spring Boot应用程序的声明式REST客户端。Feign通过可插拔的注解支持，包括Feign注解和JAX-RS注解，使编写Web服务客户端更加容易。  
此外，Spring Cloud还增加了对Spring MVC注解的支持，并支持使用与Spring Web相同的HttpMessageConverters。  
使用Spring Cloud OpenFeign的一个好处是，除了一个接口定义外，我们不需要写任何调用服务的代码。

## 与 openfeign 的区别
[openfeign](https://github.com/OpenFeign/feign) 是一个开源的项目，与spring 没有关系，主要是为了使编写 http 客户端请求代码变得更简单，而且在持续维护中。
[spring-cloud-openfeign](https://github.com/spring-cloud/spring-cloud-openfeign) 是构建在 openfeign 之上的，目标是简化在spring 生态中使用 openfeign，它使 openfeign 支持 springMVC的注解、支持负载均衡、服务发现、熔断等等spring 的功能，而且简化了配置复杂性。但是，这个项目目前已经不再维护升级了，官方建议迁移到[ Spring HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-service-client)。

下边是原生 Feign 使用的示例：
```
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);

}

public static class Contributor {
  String login;
  int contributions;
}

public static class Issue {
  String title;
  String body;
  List<String> assignees;
  int milestone;
  List<String> labels;
}

public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
						 .encoder(new GsonEncoder())
                         .target(GitHub.class, "https://api.github.com");

    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

## 集成步骤
1. 添加依赖
	````
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-openfeign</artifactId>
	</dependency>
	````
2. 接下来，我们需要将 `@EnableFeignClients` 添加到我们的主类中。有了这个注解，我们就可以对那些声明 `@FeignClient` 的接口进行组件扫描。
	```
	@SpringBootApplication
	@EnableFeignClients
	public class ExampleApplication {

		public static void main(String[] args) {
			SpringApplication.run(ExampleApplication.class, args);
		}
	}
	```
3. 然后我们使用 `@FeignClient` 注解来声明一个Feign客户端。
	```
	@FeignClient(value = "jplaceholder", url = "https://jsonplaceholder.typicode.com/")
	public interface JSONPlaceHolderClient {

		@RequestMapping(method = RequestMethod.GET, value = "/posts")
		List<Post> getPosts();

		@RequestMapping(method = RequestMethod.GET, value = "/posts/{postId}", produces = "application/json")
		Post getPostById(@PathVariable("postId") Long postId);
	}
	```
	在 `@FeignClient` 注解中的value参数是一个强制性的、任意的客户端名称，指定了当前 feign client 的名字，如果集成了服务发现，这个名字会作为服务发现的服务名去解析服务端的URL，而通过url参数，我们可以人工指定服务端API的基本URL。
4. 可以在其他Bean 中注入声明的 Feign 客户端了。

## 配置
**理解每个Feign客户端是由一组可定制的组件组成的，这一点非常重要。**
Spring Cloud使用 `FeignClientsConfiguration` 类为每个命名的客户端按需创建一个默认配置选项。主要包括下述配置项：
+ Decoder – `ResponseEntityDecoder` ，它包装了 `SpringDecoder` ，用于解码 Response 响应， 将 http 响应解码成方法的返回参数。
+ Encoder – `SpringEncoder`， 用于编码请求，将方法参数编码成 HTTP 请求参数内容。
+ Logger – 日志logger ,`Slf4jLogger` 是默认的日志。
+ MicrometerObservationCapability - micrometerObservationCapability ,如果类路径上有 `feign-micrometer` ，且 `ObservationRegistry` 可用
+ CachingCapability - 缓存，如果使用 `@EnableCaching` 注解。可通过 `spring.cloud.openfeign.cache.enabled` 停用。
+ Contract – `SpringMvcContract`, 它提供了注释处理
+ Feign.Builder – `FeignCircuitBreaker.Builder`，是用来创建 Feign 客户端组件的
+ Client – `FeignBlockingLoadBalancerClient` 或者默认的 Feign client
+ Retryer - 重试组件

等效代码
```
Feign.builder()
// Decode JSON from respone body
.decoder(new SpringDecoder(null))
// Encode JSON for request body
.encoder(new FormEncoder())
.retryer(new Retryer.Default())
.client(new OkHttpClient(new okhttp3.OkHttpClient().newBuilder()
		.connectTimeout(1, TimeUnit.MINUTES)
		.writeTimeout(1, TimeUnit.MINUTES)
		.readTimeout(1, TimeUnit.MINUTES)
		.build()))
.target(CmsService.class, "/monkeys");
```

Spring-cloud-openfeign中的自动配置类： `FeignAutoConfiguration` 。

### 自定义 Java 代码配置
如果我们想定制这些Bean中的一个或多个，我们可以通过创建一个配置类来覆盖它们，然后将其添加到 `FeignClient` 注解的 `configuration` 属性中。
```
@FeignClient(value = "jplaceholder",
  url = "https://jsonplaceholder.typicode.com/",
  configuration = ClientConfiguration.class)
public interface TestClient {
	@RequestMapping(method = RequestMethod.GET, value = "/posts")
	List<Post> getPosts();
}


public class ClientConfiguration {
    @Bean
    public OkHttpClient client() {
        return new OkHttpClient();
    }
}
```

`ClientConfiguration` 不需要注释 `@Configuration` 。但是，如果使用了 `@Configuration` ，则应注意将其排除在任何包含此配置的 `@ComponentScan` 之外，因为指定后它将成为 `feign.Decoder` 、`feign.Encoder` 、 `feign.Contract` 等的默认配置。要避免这种情况，可以将其放在与任何 `@ComponentScan` 或 `@SpringBootApplication` 无关的独立包中，或者在 `@ComponentScan` 中明确将其排除。


### 配置文件配置
也可以使用配置文件来配置Feign客户端，而不是使用配置类。

老版本的超时配置：
```
feign.httpclient.connection-timeout=2000
feign.httpclient.ok-http.read-timeout=6000
```

最新版本application.yaml例子：
```
spring:
    cloud:
        openfeign:
            client:
                config:
					default:
						connectTimeout: 5000
						readTimeout: 5000
						loggerLevel: basic
                    client1:
                        url: http://remote-service.com
                        connectTimeout: 5000
                        readTimeout: 5000
                        loggerLevel: full
                        errorDecoder: com.example.SimpleErrorDecoder
                        retryer: com.example.SimpleRetryer
                        defaultQueryParameters:
                            query: queryValue
                        defaultRequestHeaders:
                            header: headerValue
                        requestInterceptors:
                            - com.example.FooRequestInterceptor
                            - com.example.BarRequestInterceptor
                        responseInterceptor: com.example.BazResponseInterceptor
                        dismiss404: false
                        encoder: com.example.SimpleEncoder
                        decoder: com.example.SimpleDecoder
                        contract: com.example.SimpleContract
                        capabilities:
                            - com.example.FooCapability
                            - com.example.BarCapability
                        queryMapEncoder: com.example.SimpleQueryMapEncoder
                        micrometer.enabled: false
			compression:
			  	request:
					enabled: true //开启请求GZIP压缩
				response:
					enabled: true //开启响应GZIP压缩
```
可以创建以 default 为客户端名称的配置来配置所有的 `@FeignClient` 对象，我们也可以为一个声明的特定 feign 客户端(上面的client1)名称创建配置。

**如果我们同时拥有 java 配置和配置文件配置，配置文件的属性将覆盖 java 配置的值。如果我们想要java配置覆盖配置文件的配置，可以设置`spring.cloud.openfeign.client.default-to-properties=false`**

### 熔断降级
Spring Cloud CircuitBreaker 支持回退概念：当电路断开或出现错误时执行的默认代码路径。所以，
1. 第一步，需要添加依赖：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
	<version>2.1.8</version>
</dependency>
```

2. 第二步： 配置 `feign.circuitbreaker.enabled=true`，这个要根据使用的 openFeigin 版本来定，最新的版本是 `spring.cloud.openfeign.circuitbreaker.enabled=true`
3. 第三步：要为给定的 `@FeignClient` 启用回退，请将 `fallback` 属性设置为实现回退的类名。您还需要将您的实现声明为 Spring Bean。
```
@FeignClient(name = "test", url = "http://localhost:${server.port}/", fallback = Fallback.class)
protected interface TestClient {

	@RequestMapping(method = RequestMethod.GET, value = "/hello")
	Hello getHello();

	@RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
	String getException();

}

@Component
static class Fallback implements TestClient {
	@Override
	public Hello getHello() {
		throw new NoFallbackAvailableException("Boom!", new RuntimeException());
	}

	@Override
	public String getException() {
		return "Fixed response";
	}
}
```

如果需要根据异常内容来定义降级策略，可以使用 `FallbackFactory`：
```
@FeignClient(name = "testClientWithFactory", url = "http://localhost:${server.port}/",
            fallbackFactory = TestFallbackFactory.class)
protected interface TestClientWithFactory {

    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello getHello();

    @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
    String getException();

}

@Component
static class TestFallbackFactory implements FallbackFactory<FallbackWithFactory> {

    @Override
    public FallbackWithFactory create(Throwable cause) {
        return new FallbackWithFactory();
    }

}

static class FallbackWithFactory implements TestClientWithFactory {

    @Override
    public Hello getHello() {
        throw new NoFallbackAvailableException("Boom!", new RuntimeException());
    }

    @Override
    public String getException() {
        return "Fixed response";
    }

}
```

### Interceptors
如果我们需要在请求之前对request 进行特定的操作，可以使用 RequestInterceptor。
1. RequestInterceptor
	```
	@Bean
	public RequestInterceptor requestInterceptor() {
	return requestTemplate -> {
			requestTemplate.header("user", username);
			requestTemplate.header("password", password);
			requestTemplate.header("Accept", ContentType.APPLICATION_JSON.getMimeType());
		};
	}
	```
2. BasicAuthRequestInterceptor
   ```
    @Bean
	public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
		return new BasicAuthRequestInterceptor("username", "password");
	}
   ```

### 配置 okhttp 连接池
1. 添加依赖：
   ```
    <dependency>
		<groupId>io.github.openfeign</groupId>
		<artifactId>feign-okhttp</artifactId>
	</dependency>
   ```
2. 配置连接池：
   ```
	public class ChannelGatewayConfiguration {
		@Bean
		public okhttp3.OkHttpClient okHttpClient() {
			HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
			logging.setLevel(HttpLoggingInterceptor.Level.BASIC);

			return new okhttp3.OkHttpClient.Builder()
					.connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES)) // 最大50个空闲连接，保活5分钟
					.connectTimeout(10, TimeUnit.SECONDS)
					.readTimeout(20, TimeUnit.SECONDS)
					.writeTimeout(20, TimeUnit.SECONDS)
					.addInterceptor(logging)
					.build();
		}

		@Bean
		public feign.Client feignClient(okhttp3.OkHttpClient okHttpClient) {
			return new feign.okhttp.OkHttpClient(okHttpClient); // 使用 OkHttp 封装的 Feign 客户端
		}
	}
   ```

## 工作原理
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients{}
```

<center><img src="pics/feign-process-imports.png" width="60%"></center>

`FeignClientsRegistrar` 实现了 `ImportBeanDefinitionRegistrar` 接口。
由源码可以看出， `ImportBeanDefinitionRegistrar` 本质上是一个接口。在 `ImportBeanDefinitionRegistrar` 接口中，有一个 `registerBeanDefinitions()` 方法，通过 `registerBeanDefinitions()` 方法，我们可以向Spring容器中注册bean实例。

Spring官方在动态注册bean时(大多数时候是为一些特殊注解生成代理 bean)，大部分套路其实是使用 `ImportBeanDefinitionRegistrar` 接口。

所有实现了该接口的类都会被 `ConfigurationClassPostProcessor` 处理， `ConfigurationClassPostProcessor` 实现了 `BeanFactoryPostProcessor` 接口，所以 `ImportBeanDefinitionRegistrar` 中动态注册的bean是优先于依赖其的bean初始化的，也能被aop、validator等机制处理。

`FeignClientFactoryBean`
`FeignBuilderCustomizer`

`ReflectiveFeign` 的 `newInstance()` 方法
```
public <T> T newInstance(Target<T> target, C requestContext) {
	ReflectiveFeign.TargetSpecificationVerifier.verify(target);
	Map<Method, InvocationHandlerFactory.MethodHandler> methodToHandler = this.targetToHandlersByName.apply(target, requestContext);
	InvocationHandler handler = this.factory.create(target, methodToHandler);
	T proxy = Proxy.newProxyInstance(target.type().getClassLoader(), new Class[]{target.type()}, handler);
	Iterator var6 = methodToHandler.values().iterator();

	while(var6.hasNext()) {
		InvocationHandlerFactory.MethodHandler methodHandler = (InvocationHandlerFactory.MethodHandler)var6.next();
		if (methodHandler instanceof DefaultMethodHandler) {
			((DefaultMethodHandler)methodHandler).bindTo(proxy);
		}
	}

	return proxy;
}
``` 
最终返回的是一个 Proxy 对象。


## 实用技巧
1. 动态构造请求路径：
```
@PostMapping("/agreement/v0/{apiNo}")
Response<Object> agreement(GenericRequest request, @PathVariable String apiNo);
```

2. `Interceptor` 拦截请求统一处理：
```
@Bean
public RequestInterceptor interceptor() {
	return requestTemplate -> {
		Header header = GrpcContextKeys.HEADER.get();
		requestTemplate.header("x-request-id", header.getXRequestId());
		requestTemplate.header("platform", header.getPlatform());
		requestTemplate.header("osVersion", header.getOsVersion());
		requestTemplate.header("xff", header.getXff());
		requestTemplate.header("version", header.getVersion());
		requestTemplate.header("accept-language", header.getLang());
	};
}
```