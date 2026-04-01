#  Springboot 集成
{docsify-updated}
> https://resilience4j.readme.io/docs/getting-started-3

## 添加依赖
```
<dependency>
	<groupId>io.github.resilience4j</groupId>
	<artifactId>resilience4j-spring-boot3</artifactId>
	<version>${resilience4jVersion}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-aop</artifactId>
</dependency>

```

## 基于Yaml的配置
```
resilience4j.circuitbreaker:
    instances:
		default:
			slidingWindowSize: 100
			permittedNumberOfCallsInHalfOpenState: 10
			waitDurationInOpenState: 10000
			failureRateThreshold: 60
			eventConsumerBufferSize: 10
			registerHealthIndicator: true
        backendA:
            registerHealthIndicator: true
            slidingWindowSize: 100
        backendB:
            registerHealthIndicator: true
            slidingWindowSize: 10
            permittedNumberOfCallsInHalfOpenState: 3
            slidingWindowType: TIME_BASED
            minimumNumberOfCalls: 20
            waitDurationInOpenState: 50s
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
            recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
            
resilience4j.retry:
    instances:
		default:
			retry-exception-predicate=com.alibaba.demo.config.TrueExpPredict //默认对exception不retry，必需配置这个才会生效
        backendA:
            maxAttempts: 3
            waitDuration: 10s
            enableExponentialBackoff: true
            exponentialBackoffMultiplier: 2
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
        backendB:
            maxAttempts: 3
            waitDuration: 10s
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
                
resilience4j.bulkhead:
    instances:
        backendA:
            maxConcurrentCalls: 10
        backendB:
            maxWaitDuration: 10ms
            maxConcurrentCalls: 20
            
resilience4j.thread-pool-bulkhead:
  instances:
    backendC:
      maxThreadPoolSize: 1
      coreThreadPoolSize: 1
      queueCapacity: 1
      writableStackTraceEnabled: true
        
resilience4j.ratelimiter:
    instances:
        backendA:
            limitForPeriod: 10
            limitRefreshPeriod: 1s
            timeoutDuration: 0
            registerHealthIndicator: true
            eventConsumerBufferSize: 100
        backendB:
            limitForPeriod: 6
            limitRefreshPeriod: 500ms
            timeoutDuration: 3s
            
resilience4j.timelimiter:
    instances:
        backendA:
            timeoutDuration: 2s
            cancelRunningFuture: true
        backendB:
            timeoutDuration: 1s
            cancelRunningFuture: false
```

## Java 代码配置
```
@Bean
public CircuitBreakerConfigCustomizer testCustomizer() {
    return CircuitBreakerConfigCustomizer
        .of("backendA", builder -> builder.slidingWindowSize(100));
}
```

Resilience4j有自己的customizer类型，可以按上述代码示使用：

| Resilienc4j类型    | 自定义类                           |
| :----------------- | :--------------------------------- |
| Circuit breaker    | CircuitBreakerConfigCustomizer     |
| Retry              | RetryConfigCustomizer              |
| Rate limiter       | RateLimiterConfigCustomizer        |
| Bulkhead           | BulkheadConfigCustomizer           |
| ThreadPoolBulkhead | ThreadPoolBulkheadConfigCustomizer |
| Time Limiter       | TimeLimiterConfigCustomizer        |


## 注解
```
@CircuitBreaker(name = BACKEND, fallbackMethod = "fallback")
@RateLimiter(name = BACKEND)
@Bulkhead(name = BACKEND, type = Bulkhead.Type.THREADPOOL, fallbackMethod = "fallback")
@Retry(name = BACKEND)
@TimeLimiter(name = BACKEND)
public Mono<String> method(String param1) {
    return Mono.error(new NumberFormatException());
}

private Mono<String> fallback(String param1, CallNotPermittedException e) {
	System.out.println("熔断降级");
    return Mono.just("Handled the exception when the CircuitBreaker is open");
}

private Mono<String> fallback(String param1, BulkheadFullException e) {
	System.out.println("隔离降级");
    return Mono.just("Handled the exception when the Bulkhead is full");
}

private Mono<String> fallback(String param1, NumberFormatException e) {
    return Mono.just("Handled the NumberFormatException");
}

public String fallback(RequestNotPermitted throwable) {
	System.out.println("限流降级");
	return "限流";
}

private Mono<String> fallback(String param1, Exception e) {
    return Mono.just("Handled any other exception");
}
```

>注意：降级方法（fallbackMethod）应置于相同的类中，并且必须具有相同的方法签名，只可以多一个额外的异常类型的参数。


## 切面的顺序
Resilience4j的切面执行顺序：

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )
```

所以`Retry` 是最后执行.
如果需要不同的顺序，则必须使用Java代码配置而不是Spring注解，或者使用以下属性显式设置方面顺序：

```text
- resilience4j.retry.retryAspectOrder
- resilience4j.circuitbreaker.circuitBreakerAspectOrder
- resilience4j.ratelimiter.rateLimiterAspectOrder
- resilience4j.timelimiter.timeLimiterAspectOrder
```

例如要使断路器在重试完成其工作后启动，必须将`RetryAspectOrder`属性设置为比`circuitBreakerAspectOrder`值更大的值（更高的值=更高的优先级）。

```yaml
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2
```