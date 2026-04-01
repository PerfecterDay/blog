# Resilience Features
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/resilience.html

从 7.0 版本开始，Spring 框架的核心已包含通用的容错功能，特别是用于方法调用的 `@Retryable` 和 `@ConcurrencyLimit` 注解，以及对编程重试的支持。

## @Retryable
`@Retryable` 是一个注解，用于指定单个方法的重试特性（当该注解在方法级别声明时），或指定给定某个类中所有通过代理调用的方法的重试特性（当该注解在类型级别声明时）。
```
@Retryable
public void sendNotification() {
    this.jmsClient.destination("notifications").send(...);
}
```
默认情况下，方法调用在抛出任何异常时都会进行重试：首次失败后最多重试 3 次（ `maxRetries` = 3），每次重试之间间隔 1 秒。

`@Retryable` 方法至少会被调用一次，并最多重试 `maxRetries` 次，其中 `maxRetries` 表示重试的最大次数。具体来说，`总尝试次数 = 1 次初始尝试 + maxRetries 次重试` 。  
例如，如果将 `maxRetries` 设置为 4，则 `@Retryable` 方法将被调用至少 1 次，最多 5 次。

如有必要，可以针对每种方法进行具体调整——例如，通过 `includes` 和 `excludes` 属性来缩小需要重试的异常范围。提供的异常类型将与失败调用抛出的异常以及嵌套原因进行匹配。
```
@Retryable(MessageDeliveryException.class)
public void sendNotification() {
    this.jmsClient.destination("notifications").send(...);
}
```
`@Retryable(MessageDeliveryException.class)` 是 `@Retryable(includes = MessageDeliveryException.class)` 的简写形式。

对于高级用例，还可以通过 `@Retryable` 中的 `predicate` 属性指定一个自定义的 `MethodRetryPredicate` ，该谓词将根据 `Method` 和给定的 `Throwable` 来决定是否重试失败的方法调用——例如，通过检查 `Throwable` 的消息。  
自定义 `MethodRetryPredicate` 可以与 `includes` 和 `excludes` 条件结合使用；但是， `MethodRetryPredicate` 总是在 `includes` 和 `excludes` 条件应用之后才生效。

还支持指数退让算法：
```
@Retryable(
  includes = MessageDeliveryException.class,
  maxRetries = 4,
  delay = 100,
  jitter = 10,
  multiplier = 2,
  maxDelay = 1000)
public void sendNotification() {
    this.jmsClient.destination("notifications").send(...);
}
```

`@Retryable` 也适用于返回类型为响应式的响应式方法，它能为 `pipeline` 添加 `Reactor` 的重试功能：
```
@Retryable(maxRetries = 4, delay = 100)
public Mono<Void> sendNotification() {
    return Mono.from(...);
}
```

## @ConcurrencyLimit
`@ConcurrencyLimit` 是一个注解，用于指定单个方法的**并发限制**（当该注解在方法级别声明时），或指定给定类层次结构中所有通过代理调用的方法的并发限制（当该注解在类型级别声明时）。
```
@ConcurrencyLimit(10)
public void sendNotification() {
    this.jmsClient.destination("notifications").send(...);
}
```
此机制旨在防止目标资源被过多线程同时访问，其作用类似于线程池或连接池的池大小限制——当达到限制时，系统会阻止进一步的访问。

可以选择将限制值设为 1，从而有效锁定对目标 Bean 实例的访问：
```
@ConcurrencyLimit(1)
public void sendNotification() {
    this.jmsClient.destination("notifications").send(...);
}
```

这种限制在虚拟线程（Virtual Threads）中尤为有用，因为虚拟线程通常没有设置线程池限制。对于异步任务，可以通过 `SimpleAsyncTaskExecutor` 进行限制。对于同步调用，该注解通过 `ConcurrencyThrottleInterceptor` 提供了等效的行为——该拦截器自 Spring Framework 1.0 起便已提供，可用于 AOP 框架的编程操作。

`@ConcurrencyLimit` 还具有一个 `limitString` 属性，该属性支持属性占位符和 SpEL，可作为上述基于 int 的示例的替代方案。

## Enabling Resilient Methods
与 Spring 许多基于注解的核心功能一样， `@Retryable` 和 `@ConcurrencyLimit` 被设计为元数据，可以选择遵循或忽略它们。启用弹性注解处理的最便捷方式是在相应的 `@Configuration` 类上声明 `@EnableResilientMethods` 。

此外，也可以通过在上下文中定义 `RetryAnnotationBeanPostProcessor` 或 `ConcurrencyLimitBeanPostProcessor`  bean 来单独启用这些注解。

## 编程式的 Retry 支持
与 `@Retryable` 不同，后者为 `ApplicationContext` 中注册的 Bean 中的方法提供了一种声明式方法来指定重试语义，而 `RetryTemplate` 则为重试任意代码块提供了一种编程式 API。

具体来说， `RetryTemplate` 会根据配置的 `RetryPolicy` 来执行并可能重试一个 `Retryable` 操作。
```
var retryTemplate = new RetryTemplate();
retryTemplate.execute(
        () -> jmsClient.destination("notifications").send(...));


var retryTemplate = new RetryTemplate(RetryPolicy.withMaxRetries(4));
retryTemplate.execute(
        () -> jmsClient.destination("notifications").send(...));


var retryPolicy = RetryPolicy.builder()
        .includes(MessageDeliveryException.class)
        .excludes(...)
        .build();


var retryTemplate = new RetryTemplate(retryPolicy);
retryTemplate.execute(
        () -> jmsClient.destination("notifications").send(...));



var retryPolicy = RetryPolicy.builder()
        .includes(MessageDeliveryException.class)
        .maxRetries(4)
        .delay(Duration.ofMillis(100))
        .jitter(Duration.ofMillis(10))
        .multiplier(2)
        .maxDelay(Duration.ofSeconds(1))
        .build();
var retryTemplate = new RetryTemplate(retryPolicy);
retryTemplate.execute(
        () -> jmsClient.destination("notifications").send(...));
```

尽管 `RetryPolicy` 的工厂方法和构建器 API 已涵盖了大多数常见的配置场景，但仍可实现自定义的 `RetryPolicy` ，从而完全控制应触发重试的异常类型以及要使用的 `BackOff` 策略。还可以通过 `RetryPolicy.Builder` 中的 `backOff()` 方法配置自定义的 `BackOff` 策略。