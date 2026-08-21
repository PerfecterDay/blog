# Kafka 消费常见问题
{docsify-updated}

## Poison Pill：某条消息消费一直失败会不会阻塞后续消息？

这是 Kafka 消费者最经典的**"毒丸消息"（Poison Pill）** 问题。答案取决于**错误处理方式**。

### 一、默认情况：会一直卡住，后面消息也消费不到

**Kafka 是按分区顺序消费的**，一个分区内的消息必须按 offset 递增消费。如果代码里对"某条消息一直抛异常，不 commit offset"，会发生：

```java
// 简化的默认行为
while (true) {
    ConsumerRecords<K,V> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<K,V> record : records) {
        process(record);            // ← 这里对某条消息一直抛异常
        consumer.commitSync();      // 永远走不到
    }
}
```

结果：
- **同一批 `poll()` 到的后续消息也不会被处理**（`for` 循环直接被异常打断）
- 下次 `poll()` 时，因为 offset 没提交，Kafka 会**从上一次已提交的位置重新拉取**——又是同一条毒丸消息
- **该分区完全阻塞**，后续所有消息都无法消费

⚠️ **注意分区隔离**：其他分区不受影响。同一个 consumer group 下别的 partition 的消费者线程照常工作。

### 二、Spring Kafka 的实际行为（大多数人踩坑的地方）

如果用 `@KafkaListener`，Spring 默认使用 **`DefaultErrorHandler`**（早期版本叫 `SeekToCurrentErrorHandler`），行为是：

1. 消息处理抛异常
2. **在内存里**做几次重试（默认 `FixedBackOff(0L, 9)`，即立刻重试 9 次）
3. 重试用完还失败 → **调用 `recoverer`**，默认 recoverer 是 `LoggingRecoverer`（只打日志！）
4. 打完日志后 **`seek` 到该消息之后**，接着消费下一条

⚠️ 关键的坑：**默认 recoverer 只是打日志，会把这条毒丸消息"跳过"，后续消息能继续消费，但这条消息本身就丢了（业务上没处理）**。很多人以为"手动提交就不会丢"，其实默认配置下 **跳过 = 丢消息**。

### 三、正确的做法：三选一

#### 方案 A：Dead Letter Topic（推荐）

重试 N 次仍失败 → 把消息扔到死信 topic，然后正常前进：

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> tpl) {
    var recoverer = new DeadLetterPublishingRecoverer(tpl,
        (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition()));

    var backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000);
    backoff.setMultiplier(2.0);
    backoff.setMaxInterval(10000);

    var handler = new DefaultErrorHandler(recoverer, backoff);
    // 业务异常不重试直接进 DLT
    handler.addNotRetryableExceptions(IllegalArgumentException.class);
    return handler;
}
```

效果：
- 毒丸消息重试 3 次都失败 → 自动发送到 `原topic.DLT`
- 主流程 offset 前进，**后续消息正常消费**
- 有单独的消费者/人工处理 DLT，避免消息真正丢失

#### 方案 B：StopContainer（保数据完整性优先）

对数据严格的场景（比如金融），宁可阻塞也不能跳过：

```java
var handler = new DefaultErrorHandler(
    new CommonContainerStoppingErrorHandler(),   // recoverer：停容器
    new FixedBackOff(5000L, 3));
```

重试用完后**停掉 listener container**，业务人员介入修复。这时该分区确实卡住，但保证不丢数据。

#### 方案 C：业务层 try-catch + 手动 ack

最灵活但也最容易出错：

```java
@KafkaListener(topics = "orders", ackMode = MANUAL)
public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        process(record);
    } catch (BusinessException e) {
        // 业务错误：写到自己的错误表 or 发到 DLT
        errorRepo.save(new FailedMessage(record, e));
    } catch (Exception e) {
        // 系统错误：抛出去让 ErrorHandler 处理（重试）
        throw e;
    }
    ack.acknowledge();   // ← 一定要 ack，否则后面消息全卡住
}
```

关键点：**业务错误自己吞掉并落库，只有可重试的系统错误才抛出去**。

### 四、还有一个关键点：`max.poll.interval.ms`

如果你一直在处理同一条消息（比如无限重试）：

```yaml
spring:
  kafka:
    consumer:
      properties:
        max.poll.interval.ms: 300000   # 默认 5 分钟
```

处理时间超过这个值，**Kafka 会认为这个消费者挂了**，触发 rebalance，你的分区被分给别的实例，然后循环重演……最终整个 group 都卡死。

### 五、行为对照表

| 场景 | 该分区后续消息能被消费吗 | 其他分区呢 |
|---|---|---|
| 无 error handler，代码 catch 后不 ack | ❌ 无限重放同一条 | ✅ 正常 |
| 无 error handler，代码不 catch 直接抛 | ❌ 无限重放同一条 | ✅ 正常 |
| Spring 默认 `DefaultErrorHandler`（只 log） | ✅ 能，但**毒丸消息丢失** | ✅ 正常 |
| `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` | ✅ 能，毒丸进 DLT | ✅ 正常 |
| `DefaultErrorHandler` + `ContainerStoppingErrorHandler` | ❌ 容器停止 | ✅ 正常 |
| 业务层 try-catch，异常吞掉 + 手动 ack | ✅ 能，但要保证异常消息落库 | ✅ 正常 |
| 一直重试超过 `max.poll.interval.ms` | ❌ 触发 rebalance 恶性循环 | ⚠️ 可能被拖累 |

### 六、一句话结论

> **一条消息 offset 没提交，同分区后面的消息不会被消费**（Kafka 分区内严格有序，必须按 offset 递增前进）。但是：
> - **其他分区不受影响**
> - 通过 **DLT + 重试 backoff** 可以让主流程越过毒丸继续消费
> - 不能只靠"手动提交 offset"这一个机制来解决这个问题，**必须配上 error handler + 死信策略**

### 七、生产级最佳实践

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      max-poll-records: 50
      max-poll-interval-ms: 300000
      properties:
        isolation.level: read_committed
    listener:
      ack-mode: MANUAL_IMMEDIATE      # 手动提交，处理完立即 commit
      type: SINGLE                    # 一条一条处理，简化错误定位
```

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<?,?> tpl) {
    var recoverer = new DeadLetterPublishingRecoverer(tpl,
        (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition()));
    var backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000);
    backoff.setMultiplier(2.0);
    var h = new DefaultErrorHandler(recoverer, backoff);
    // 反序列化异常这种"永远修不好"的直接进 DLT，不要重试
    h.addNotRetryableExceptions(DeserializationException.class,
                                 MessageConversionException.class,
                                 IllegalArgumentException.class);
    return h;
}

@Bean
public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>>
kafkaListenerContainerFactory(ConsumerFactory<String, String> consumerFactory, DefaultErrorHandler errorHandler) {
    ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

    // 注入消费者工厂
    factory.setConsumerFactory(consumerFactory);
    // 配置手动提交偏移量，与yml配置 ack-mode: manual 完全一致
    factory.getContainerProperties().setAckMode(ackMode);
    // 注入错误处理器，消费失败时按策略重试后放入死信队列
    factory.setCommonErrorHandler(errorHandler);

    // 可选配置：设置消费并发数，建议值 ≤ Topic分区数量，根据业务调整
    // factory.setConcurrency(3);

    return factory;
}

@KafkaListener(topics = "orders")
public void listen(OrderEvent evt, Acknowledgment ack) {
    // 消费幂等：先查处理表
    if (processedRepo.existsById(evt.getId())) {
        ack.acknowledge();
        return;
    }
    process(evt);                                          // 业务处理
    processedRepo.save(new Processed(evt.getId()));       // 幂等标记
    ack.acknowledge();
}
```

再配一个 DLT 消费者，负责告警 + 人工/自动重放，就构成完整闭环。

