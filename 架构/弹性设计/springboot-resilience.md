# Spring Boot 应用的弹性设计
{docsify-updated}

> 在 Spring Boot 项目里，**外部 API 调用、数据库操作、Kafka 消息发送** 是最常见的三类外部依赖。它们的失败模式和应对手段各不相同，需要分别设计 resilience 策略。

---

## 一、整体思路：三类操作的 Resilience 关注点不一样

| 操作类型 | 主要风险 | 主要手段 |
|---|---|---|
| 外部 API 调用 | 慢、超时、对方挂了拖死自己 | **超时 + 熔断 + 重试 + 降级 + Bulkhead** |
| 数据库操作 | 连接耗尽、慢查询、死锁 | **连接池限流 + 超时 + 重试（仅幂等）+ 事务** |
| Kafka 发消息 | 与 DB 双写不一致、Broker 抖动 | **Producer 重试 + Outbox + 幂等 Consumer + DLQ** |

最关键的一句话：**重试要分场景**——读操作或天然幂等的写可以重试；非幂等写绝不能盲目重试，要么先加幂等键，要么用 Outbox 兜底。

> 相关笔记：
> - [Spring 6 Resilience 支持（@Retryable / @ConcurrencyLimit）](/spring/core/Resilience支持.md)
> - [resilience4j](/resilience4j/_sidebar.md)
> - [双写问题及解决方案](/架构/双写问题/dual-writes.md)
> - [Outbox 模式](/架构/双写问题/outbox.md)
> - [DB 与 Redis 一致性](/架构/双写问题/db-redis一致性.md)

---

## 二、外部 API 调用

### 2.1 客户端基础设施：先把超时配死

用 Spring 6 的 `RestClient` / `WebClient` 时，要先把"连接 + 读"超时配死，否则后面所有 resilience 都失效：

```java
@Bean
RestClient restClient() {
    var factory = new JdkClientHttpRequestFactory();
    factory.setReadTimeout(Duration.ofSeconds(2));
    return RestClient.builder()
        .requestFactory(factory)
        .baseUrl("https://upstream")
        .build();
}
```

> **没有超时的客户端 = 没有 resilience。** 这是最常见的事故根源。

### 2.2 Resilience4j 三件套（推荐组合）

`application.yml`：

```yaml
resilience4j:
  timelimiter:
    instances:
      paymentApi:
        timeoutDuration: 2s
        cancelRunningFuture: true
  circuitbreaker:
    instances:
      paymentApi:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 50
        minimumNumberOfCalls: 20
        failureRateThreshold: 50           # 50% 失败就熔断
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
  retry:
    instances:
      paymentApi:
        maxAttempts: 3
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - org.springframework.web.client.ResourceAccessException
        ignoreExceptions:
          - com.example.BusinessException     # 业务异常不重试
  bulkhead:
    instances:
      paymentApi:
        maxConcurrentCalls: 30               # 隔离上游慢调用
        maxWaitDuration: 0
```

Java 端使用注解（**注意顺序**：外层是 Retry → CircuitBreaker → TimeLimiter）：

```java
@Service
public class PaymentClient {

    @TimeLimiter(name = "paymentApi")
    @CircuitBreaker(name = "paymentApi", fallbackMethod = "fallback")
    @Retry(name = "paymentApi")
    @Bulkhead(name = "paymentApi")
    public CompletableFuture<PayResp> pay(PayReq req) {
        return CompletableFuture.supplyAsync(() -> restClient.post()...);
    }

    private CompletableFuture<PayResp> fallback(PayReq req, Throwable t) {
        // 1) 返回降级响应；2) 进入异步重试队列；3) 抛业务异常让上游拒绝
        return CompletableFuture.completedFuture(PayResp.degraded());
    }
}
```

### 2.3 关键经验

- **超时必须 < 熔断慢调用阈值 < 上游 SLA**。
- **熔断器要按依赖维度拆分**（每个上游一个实例），不要全局一个。
- **重试只对网络抖动类异常**（IOException、5xx、连接异常），4xx 一般不重试。
- **POST/PUT 这种非幂等接口**：要么让上游支持幂等键（`Idempotency-Key` Header），要么不要重试。
- **Bulkhead 防止"一个慢上游耗尽所有线程"**。Spring Boot 默认 Tomcat 线程池被打满后整个应用都挂了。
- Spring 6 的 [`@Retryable`](/spring/core/Resilience支持.md) 也可以作为更轻量的替代（仅重试，不带熔断）。

---

## 三、数据库操作

### 3.1 连接池层（HikariCP）

数据库的 resilience 第一道防线是**连接池配置**，不是注解：

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000        # 拿不到连接 3s 就失败，防雪崩
      validation-timeout: 1000
      max-lifetime: 1800000           # 小于 DB/中间件的 idle timeout
      keepalive-time: 300000
      leak-detection-threshold: 30000
```


JDBC 层超时：

```yaml
spring:
  jpa:
    properties:
      javax.persistence.query.timeout: 3000     # 单条 SQL 3s
      jakarta.persistence.lock.timeout: 2000
```

> 没有 `connection-timeout` 和 `query timeout`，DB 抖动一次整个应用就挂。

### 3.2 事务级别的重试（仅幂等）

只对**可重试的失败**重试，例如乐观锁冲突、序列化失败、死锁：

```java
@Retryable(
    include = { OptimisticLockingFailureException.class,
                CannotAcquireLockException.class },
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2, maxDelay = 1000))
@Transactional
public Order placeOrder(OrderCmd cmd) { ... }
```

⚠️ **重试一定要在事务方法外层，或重新开新事务**。已经回滚的事务在同一个 `@Transactional` 里再操作会拿到 `rollback-only` 错误。

### 3.3 慢查询保护

- 给所有查询加 SQL `statement_timeout`（PG）或 `MAX_EXECUTION_TIME`（MySQL hint）。
- 长事务拆短；只读查询用 `@Transactional(readOnly = true)`。
- 写操作前先用唯一键 / `INSERT ... ON CONFLICT` 做幂等。

### 3.4 不要给 DB 调用包熔断器

这是一个常见误区。**数据库通常是单点依赖，熔断了应用就不可用**，且 DB 慢通常是连接池满或 SQL 慢导致的，应该让请求自然超时排队，而不是把所有写入丢弃。

正确做法：**`HikariCP.connection-timeout` + `query timeout` + 慢查询监控告警**。

---

## 四、Kafka 发送消息（最复杂的一类）

参考 [Outbox 模式](/架构/双写问题/outbox.md) 和 [双写问题及解决方案](/架构/双写问题/dual-writes.md)，这里直接给落地方案。

### 4.1 Producer 端基本配置

```yaml
spring:
  kafka:
    producer:
      acks: all                          # 强一致
      retries: 2147483647                # 重试到天荒地老（配合 max.block.ms）
      properties:
        max.block.ms: 5000               # 元数据获取/缓冲区等待最多 5s
        request.timeout.ms: 10000
        delivery.timeout.ms: 30000       # 发送总超时
        enable.idempotence: true         # 启用幂等 producer，避免重复
        max.in.flight.requests.per.connection: 5
        linger.ms: 5
        compression.type: lz4
```

`enable.idempotence: true` 让 Kafka 自己做去重（基于 PID + Sequence Number），是必开项。

### 4.2 最大的坑：DB 和 Kafka 的双写一致性

```java
// ❌ 反例：典型双写问题
@Transactional
public void createOrder(OrderCmd cmd) {
    orderRepo.save(order);                        // 1. DB
    kafkaTemplate.send("order.created", event);   // 2. Kafka
}
```

问题：
- DB 提交后 Kafka 发送失败 → 消息丢失
- Kafka 发送成功后 DB 回滚 → 假消息
- 即使两个都"成功"，事务里 Kafka 走的是异步 send，回调晚于事务提交

### 4.3 正确做法：Transactional Outbox

只写一张 outbox 表，单独一个 worker 异步投递到 Kafka：

```java
@Transactional
public void createOrder(OrderCmd cmd) {
    orderRepo.save(order);

    // 同一个事务里只动 DB
    outboxRepo.save(new OutboxMessage(
        "order.created",
        order.getId().toString(),
        objectMapper.writeValueAsString(event)
    ));
}
```

后台 worker：

```java
@Scheduled(fixedDelay = 500)
public void publish() {
    List<OutboxMessage> batch = outboxRepo.findUnpublished(Limit.of(100));
    for (OutboxMessage m : batch) {
        try {
            kafkaTemplate.send(m.getTopic(), m.getKey(), m.getPayload())
                .get(5, TimeUnit.SECONDS);
            outboxRepo.markPublished(m.getId());
        } catch (Exception e) {
            outboxRepo.incrementRetry(m.getId());
            if (m.getRetryCount() >= 5) moveToDeadLetter(m);
        }
    }
}
```

进阶可以用 **Debezium / Canal 订阅 outbox 表的 binlog**，省掉轮询。详见 [Outbox 模式](/架构/双写问题/outbox.md)。

### 4.4 Consumer 端的 Resilience

- **手动 commit + 至少一次**：处理成功后再 commit offset，配合幂等消费。
- **DLT（死信主题）**：Spring Kafka 自带 `DefaultErrorHandler + DeadLetterPublishingRecoverer`：

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<?,?> tpl) {
    var recoverer = new DeadLetterPublishingRecoverer(tpl,
        (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition()));
    var backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(500);
    backoff.setMultiplier(2.0);
    return new DefaultErrorHandler(recoverer, backoff);
}
```

- **消费幂等**：用消息里的业务 ID + 一张 `processed_event(event_id PK)` 表，或 Redis SETNX 去重。
- **隔离慢消费者**：单独 listener container + 单独线程池，不和 HTTP 共用 Tomcat 线程。

---


## 五、把它们粘起来：一个真实下单接口

```java
@RestController
public class OrderController {

    @PostMapping("/orders")
    public OrderResp create(@RequestBody @Idempotent OrderCmd cmd) {
        // ① 风控调外部 API：CircuitBreaker + TimeLimiter + Retry + Fallback
        var risk = riskClient.check(cmd);          // 失败降级为"放行 + 异步审计"

        // ② DB 写入 + outbox（同一事务）
        var order = orderAppService.placeOrder(cmd);

        // ③ outbox worker 异步把 order.created 推到 Kafka
        return OrderResp.from(order);
    }
}
```

关键 resilience 点：
1. `@Idempotent`：请求级幂等（基于 Header 的 `Idempotency-Key` + Redis）
2. `riskClient`：Resilience4j 隔离上游慢调用 + 降级
3. `placeOrder`：事务里只写 DB + outbox
4. outbox worker：保证消息最终到达 Kafka
5. 下游 consumer：DLT + 幂等表 + 手动 ack

---

## 六、可观测性（Resilience 的前提）

没有指标的 resilience 是盲调。最少要打的指标：

- **Resilience4j**：`resilience4j_circuitbreaker_state`、`resilience4j_retry_calls`、`resilience4j_bulkhead_available_concurrent_calls`
- **HikariCP**：`hikaricp_connections_active`、`hikaricp_connections_pending`、`hikaricp_connections_timeout`
- **Kafka Producer**：`kafka_producer_record_error_total`、`record_send_total`、`outgoing_byte_rate`
- **Outbox 表**：`outbox_pending_count`、`outbox_oldest_age_seconds`（最重要的告警指标）

配合 Spring Boot Actuator + Micrometer + Prometheus / SkyWalking 即可。

---

## 七、一张速查表

| 场景 | 怎么做 | 避免什么 |
|---|---|---|
| 调外部 HTTP API | TimeLimiter + CircuitBreaker + Retry + Bulkhead + Fallback | 没超时、全局共用一个熔断器、对 4xx 重试 |
| 调外部 RPC（gRPC） | Deadline + 客户端负载均衡 + Resilience4j | 不设 deadline |
| DB 读 | 连接池超时 + 查询超时 + 只读事务 | 长事务、N+1 |
| DB 写 | 短事务 + 乐观锁/唯一键 + 选择性重试 | 给 DB 加熔断器；非幂等盲目重试 |
| 同事务里同时写 DB+Kafka | **Outbox** | 直接 send；@Transactional 里 send |
| Kafka 发送 | 幂等 Producer + acks=all + 重试 | 单条 send 后不等回调就 commit |
| Kafka 消费 | 手动 ack + DLT + 幂等表 + ExponentialBackOff | 自动 ack、catch 后吞异常 |
| 定时任务 | ShedLock + 分页 + 限速 | 整批一次性处理 |

---

## 八、一句话总结

> **外部 API 用 Resilience4j 保护自己不被拖死；数据库用连接池/超时/选择性重试保护数据库不被打挂；Kafka 用 Outbox + 幂等保证消息最终一致。** 三者结合，再加上指标告警，就构成了 Spring Boot 应用工业级的 resilience 体系。
