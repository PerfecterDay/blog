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