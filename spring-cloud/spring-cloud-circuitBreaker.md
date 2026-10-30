# spring-cloud-circuitBreaker
{docsify-updated}

> https://docs.spring.io/spring-cloud-circuitbreaker/reference/index.html

## 核心抽象
+ `CircuitBreakerFactory/AbstractCircuitBreakerFactory` 抽象类: 用于创建 `CircuitBreaker` 实例的工厂。它提供了一种机制，用于根据配置生成断路器实例。
  ```
  	public abstract class CircuitBreakerFactory<CONF, CONFB extends ConfigBuilder<CONF>>
		extends AbstractCircuitBreakerFactory<CONF, CONFB> {
		public abstract CircuitBreaker create(String id);
		public CircuitBreaker create(String id, String groupName) {
			return create(id);
		}
	}
  ```
+ `CircuitBreaker` 接口 : 定义了断路器模式的接口。它提供了一种机制，用于执行受断路器保护的逻辑。
+ `ConfigBuilder` 接口 : 用于构建断路器配置的接口。它提供了一种机制，用于生成配置断路器的配置类实例。
+ `Customizer` 接口 : 用于自定义断路器的接口。它提供了一种机制，用于在创建断路器之前修改其配置。

这里有一个重要的模式： `ConfigBuilder` 生成 `CONF` 配置类。后面会看到 `Resilience4J` 的实现中有多处有这种设计。
+ `Resilience4JConfigBuilder --> Resilience4JConfigBuilder.Resilience4JCircuitBreakerConfiguration`
+ `TimeLimiterConfig.Builder --> TimeLimiterConfig`
+ `CircuitBreakerConfig.Builder --> CircuitBreakerConfig`

注意，builder 和 对应的 CONF 类总是内部类形式出现，只是有时候 builder 是静态内部类，有时候 CONF 是内部类。

## Configuring Resilience4J Circuit Breakers

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

可以通过将 `spring.cloud.circuitbreaker.resilience4j.enabled` 设置为 `false` 来禁用 Resilience4J 的自动配置.

### 自动配置
`Resilience4JAutoConfiguration` 实现了自动配置功能。

### 编程式自定义配置
`Resilience4JCircuitBreakerFactory` 提供了多个配置方法：
+ `configureDefault` : 为所有断路器配置默认配置。
+ `configure` : 为特定的断路器配置个性化配置。
+ `addCircuitBreakerCustomizer` : 为特定的断路器添加自定义器。
+ `configureExecutorService` : 配置线程池。

#### 配置默认配置
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
	return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
			.timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build())
			.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
			.build());
}
```

#### 配置线程池
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
	return factory -> {
		ContextAwareScheduledThreadPoolExecutor executor = ContextAwareScheduledThreadPoolExecutor.newScheduledThreadPool().corePoolSize(5)
			.build();
		factory.configureExecutorService(executor);
	};
}
```

#### 指定不同的个性化配置
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
	return factory -> {
		factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
				.timeLimiterConfig(TimeLimiterConfig.custom()
						.timeoutDuration(Duration.ofSeconds(6)).build()), "pay", "slowgroup");

		factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
				.timeLimiterConfig(TimeLimiterConfig.custom()
						.timeoutDuration(Duration.ofSeconds(1)).build()), "order", "fastgroup");
		factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults()));
	};
}
```

`CircuitBreakerConfig.Builder` 能进行更错的配置，可以查看 `CircuitBreakerConfig` 的 `static` 方法。


以上配置在创建断路器时，会根据断路器的 id 和 groupName 选择合适的配置。如果想让控制断路器运行时的一些行为，比如添加事件处理程序，可以在断路器创建完成但尚未返回给调用方之前对其进行自定义。
要实现这一点，可以使用 `addCircuitBreakerCustomizer` 方法。这对于向 `Resilience4J` 断路器添加事件处理程序非常有用。这段代码会在 `CircuitBreaker.run()` 方法执行之前添加事件处理程序。
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
	return factory -> factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
	.onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
}
```
**简单点说，就是在 `addCircuitBreakerCustomizer` 方法中能够获取到创建完成的 `CircuitBreaker` 实例，然后可以对它进行自定义配置。**  之前的配置这些都是在 `Resilience4JCircuitBreakerFactory.create()` 之前控制的，这个是在 `create()` 之后对已经创建完成的 `CircuitBreaker` 实例进行自定义配置。

### 配置文件配置
可以在应用程序的配置属性文件中配置 `CircuitBreaker` 和 `TimeLimiter` 的配置或实例。属性配置的优先级高于 Java `Customizer` 配置。

配置优先级从高到低为：
+ D配置：Method(id) config - on specific method or operation I
+ group 配置：Service(group) config - on specific application service or operations group 配置
+ 默认配置：Global default config

```
resilience4j.circuitbreaker:
    configs:
        default:
            registerHealthIndicator: true
            slidingWindowSize: 50
		groupA:
            registerHealthIndicator: true
            slidingWindowSize: 200
	instances:
		backendA:
         	registerHealthIndicator: true
         	slidingWindowSize: 100
     	backendB:
         	registerHealthIndicator: true
         	slidingWindowSize: 10
         	permittedNumberOfCallsInHalfOpenState: 3
         	slidingWindowType: TIME_BASED
         	recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate

resilience4j.timelimiter:
    configs:
        default:
            timeoutDuration: 5s
            cancelRunningFuture: true
		groupC:
            timeoutDuration: 3s
            cancelRunningFuture: true
	instances:
		backendA:
			timeoutDuration: 2s
			cancelRunningFuture: true
		backendB:
			timeoutDuration: 1s
			cancelRunningFuture: false
```
+ `Resilience4JCircuitBreakerFactory.create("backendA")` will apply instances backendA properties
+ `Resilience4JCircuitBreakerFactory.create("backendA", "groupA")` will apply instances backendA properties
+ `Resilience4JCircuitBreakerFactory.create("backendC")` will apply global default properties
+ `Resilience4JCircuitBreakerFactory.create("backendC", "groupC")` will apply global default CircuitBreaker properties and config groupC TimeLimiter properties

#### 禁用 TimeLimiter
```
spring:
	cloud:
		circuitbreaker:
			resilience4j:
				disable-time-limiter: true
```

以上配置会禁用所有 TimeLimiter 配置，如果需要禁用特定的 TimeLimiter，可以使用以下配置：
```
spring:
	cloud:
		circuitbreaker:
			resilience4j:
				disable-time-limiter-map:
					group1: true
					instanceA: false
					instanceB: true
```
禁用优先级顺序为：instance > group > global。根据上述配置，instanceB 和 group1 的 TimeLimiter 已禁用，但 instanceA 的 TimeLimiter 仍处于启用状态。对于所有其他实例和组，将回退到全局设置的 spring.cloud.circuitbreaker.resilience4j.disable-time-limiter 值。如果未设置 spring.cloud.circuitbreaker.resilience4j.disable-time-limiter，则默认情况下，其余断路器实例和组的 TimeLimit 功能处于启用状态。

### Bulkhead 支持
