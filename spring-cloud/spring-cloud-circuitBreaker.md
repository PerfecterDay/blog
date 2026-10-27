# spring-cloud-circuitBreaker
{docsify-updated}

> https://docs.spring.io/spring-cloud-circuitbreaker/reference/index.html


## Configuring Resilience4J Circuit Breakers

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

可以通过将 `spring.cloud.circuitbreaker.resilience4j.enabled` 设置为 `false` 来禁用 Resilience4J 的自动配置.

### 默认配置

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

### 指定不同的个性化配置
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
	return factory -> factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
			.timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(2)).build()), "slow");
}
```

除了配置已创建的断路器外，还可以在断路器创建完成但尚未返回给调用方之前对其进行自定义。要实现这一点，可以使用 `addCircuitBreakerCustomizer` 方法。这对于向 Resilience4J 断路器添加事件处理程序非常有用。这段代码会在 `CircuitBreaker.run()` 方法执行之前添加事件处理程序。
```
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
	return factory -> factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
	.onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
}
```
