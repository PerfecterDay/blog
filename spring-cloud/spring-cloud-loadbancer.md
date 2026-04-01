# spring cloud loadbancer
{docsify-updated}

> https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html  
> https://12factor.net/

Spring Cloud 提供了专属的客户端负载均衡器抽象层与实现方案。在负载均衡机制方面，新增了 `ReactiveLoadBalancer` 接口，并为其提供了基于轮询（Round-Robin）和随机（Random）的实现方案。为获取可供选择的服务实例，采用了响应式 `ServiceInstanceListSupplier` 。当前支持基于服务发现的 `ServiceInstanceListSupplier` 实现，该实现通过类路径中可用的 `DiscoveryClient` 实现从服务发现系统中获取可用实例。

## 与各种客户端集成使用
> https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/common-abstractions.html#rest-template-loadbalancer-client

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

为了便于使用 Spring Cloud LoadBalancer，我们提供了 `ReactorLoadBalancerExchangeFilterFunction` （可与 `WebClient` 一起使用）和 `BlockingLoadBalancerClient` （可与 `RestTemplate` 和 `RestClient` 一起使用）。使用示例：
+ [Spring RestTemplate as a LoadBalancer Client](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/common-abstractions.html#rest-template-loadbalancer-client)
+ [Spring RestClient as a LoadBalancer Client](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/common-abstractions.html#rest-client-loadbalancer-client)
+ [Spring WebClient as a LoadBalancer Client](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/common-abstractions.html#webclinet-loadbalancer-client)
+ [Spring WebFlux WebClient with ReactorLoadBalancerExchangeFilterFunction](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/common-abstractions.html#webflux-with-reactive-loadbalancer)

```
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

	@LoadBalanced
    @Bean
    RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }

	@Bean
	@LoadBalanced
	public WebClient.Builder loadBalancedWebClientBuilder() {
		return WebClient.builder();
	}

	@Bean
	@LoadBalanced
	RestTemplateBuilder loadBalancedRestTemplateBuilder() {
		return new RestTemplateBuilder();
	}
	
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
```

## 负载均衡器上下文的预加载
Spring Cloud LoadBalancer 为每个服务 ID 创建独立的 Spring 子上下文。默认情况下，这些上下文采用**延迟初始化机制，仅在首次对该服务 ID 进行负载均衡时才进行初始化**。

我们也可以选择**立即加载**这些上下文。通过 `spring.cloud.loadbalancer.eager-load.clients` 属性指定需要立即加载的服务 ID，例如：
```
spring.cloud-loadbalancer.eager-load.clients[0]=my-first-client
spring.cloud-loadbalancer.eager-load.clients[1]=my-second-client
```

## 自定义负载均衡策略
默认使用的 `ReactiveLoadBalancer` 实现是 `RoundRobinLoadBalancer` 。要为选定的服务或所有服务切换到不同的实现，可以使用自定义负载平衡器配置机制。
```
public class CustomLoadBalancerConfiguration {

	@Bean
	ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
			LoadBalancerClientFactory loadBalancerClientFactory) {
		String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
		return new RandomLoadBalancer(loadBalancerClientFactory
				.getLazyProvider(name, ServiceInstanceListSupplier.class),
				name);
	}
}
```

## 缓存
除了每次需要选择实例时通过 `DiscoveryClient` 获取实例的基本 `ServiceInstanceListSupplier` 实现外，我们还提供了两种缓存实现。

### 基于 Caffine 缓存的实现
如果 classpath 中包含 `com.github.ben-manes.caffeine:caffeine` ，则将使用基于 `Caffeine` 的实现。 `LoadBalancerCacheConfiguration` 可以用来配置。

### 默认的实现
如果 classpath 中没有 Caffeine，则将使用 Spring-cloud-starter-loadbalancer 自动提供的 `DefaultLoadBalancerCache` 。也是通过 `LoadBalancerCacheConfiguration` 配置

## 权重缓存配置
```
public class CustomLoadBalancerConfiguration {

	@Bean
	public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
			ConfigurableApplicationContext context) {
		return ServiceInstanceListSupplier.builder()
					.withDiscoveryClient()
					.withWeighted(instance -> ThreadLocalRandom.current().nextInt(1, 101))
					.withCaching()
					.build(context);
	}
}
```

## 基于 Zone 的配置
为实现基于区域的负载平衡，我们提供了 `ZonePreferenceServiceInstanceListSupplier` 。首先我们必须为客户端 `DiscoveryClient` 配置特定的区域 zone（例如 `eureka.instance.metadata-map.zone`），然后负载均衡会用这个值过滤可用服务实例，服务实例也必须配置 zone 参数。

`ZonePreferenceServiceInstanceListSupplier` 会过滤检索到的实例，只返回同一区域内的实例。如果区域为空或同一区域内没有实例，则会返回所有检索到的实例。

```
public class CustomLoadBalancerConfiguration {

	@Bean
	public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
			ConfigurableApplicationContext context) {
		return ServiceInstanceListSupplier.builder()
					.withDiscoveryClient()
                    .withCaching()
					.withZonePreference()
					.build(context);
	}
}
```

## 健康检查
`HealthCheckServiceInstanceListSupplier` 提供了健康检查相关的功能与配置。如果使用了服务发现，可以不用健康检查，因为服务注册发现组件已经提供了健康检查功能。

```
public class CustomLoadBalancerConfiguration {

	@Bean
	public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
			ConfigurableApplicationContext context) {
		return ServiceInstanceListSupplier.builder()
					.withDiscoveryClient()
					.withHealthChecks()
					.build(context);
	    }
}
```

## 自定义配置
使用 `@LoadBalancerClient` 自定义设置负载均衡客户端， `value` 指定要要调用的服务实例（服务发现）， `configuration` 指定自定义的配置类。
```
@Configuration
@LoadBalancerClients({@LoadBalancerClient(value = "stores", configuration = StoresLoadBalancerClientConfiguration.class), @LoadBalancerClient(value = "customers", configuration = CustomersLoadBalancerClientConfiguration.class)})
public class MyConfiguration {

	@Bean
	@LoadBalanced
	public WebClient.Builder loadBalancedWebClientBuilder() {
		return WebClient.builder();
	}
}
```

## 转换 HTTP 请求
对于 `RestTemplate` 和 `RestClient` ，您需要按以下方式实现并定义 `LoadBalancerRequestTransformer` ：
```
@Bean
public LoadBalancerRequestTransformer transformer() {
	return new LoadBalancerRequestTransformer() {
		@Override
		public HttpRequest transformRequest(HttpRequest request, ServiceInstance instance) {
			return new HttpRequestWrapper(request) {
				@Override
				public HttpHeaders getHeaders() {
					HttpHeaders headers = new HttpHeaders();
					headers.putAll(super.getHeaders());
					headers.add("X-InstanceId", instance.getInstanceId());
					return headers;
				}
			};
		}
	};
}
```

对于 WebClient，您需要按以下方式实现并定义 `LoadBalancerClientRequestTransformer` ：
```
@Bean
public LoadBalancerClientRequestTransformer transformer() {
	return new LoadBalancerClientRequestTransformer() {
		@Override
		public ClientRequest transformRequest(ClientRequest request, ServiceInstance instance) {
			return ClientRequest.from(request)
					.header("X-InstanceId", instance.getInstanceId())
					.build();
		}
	};
}
```
若定义了多个转换器，它们将按Beans定义的顺序应用。或者可以使用 `LoadBalancerRequestTransformer.DEFAULT_ORDER` 或 `LoadBalancerClientRequestTransformer.DEFAULT_ORDER` 来指定顺序。