#  Kafka-consumer 开发 
{docsify-updated}

- [Kafka-consumer 开发](#kafka-consumer-开发)
    - [消费者组与独立消费者](#消费者组与独立消费者)
    - [构建 consumer](#构建-consumer)
    - [Consumer 的主要参数](#consumer-的主要参数)
    - [重平衡](#重平衡)


### 消费者组与独立消费者
消费者组使用一个消费组名(group.id)来标记自己， topic 的每条消息都会被发送到每个订阅它的消费者组的一个消费者实例上：
1. 一个 group.id 唯一标识一个 consumer group 。一个 consumer group 可能有若干个 consumer 实例
2. 对于同一个 group 而言， topic 的每条消息只能被发送到 group 内的一个实例上
3. topic 消息可以被发送到多个 group 中。
4. 对某个 group 而言，订阅 topic 的每个分区只能分配给该 group 下的一个 consumer 实例（当然该分区还可以被分配给其他订阅该 topic 的消费者组）。

所以 Kafka 使用下面的两种方式来支持的基于队列和基于发布/订阅的两种消息模型：
1. 所有的 consumer 实例都划分到同一个 group -> 实现基于队列的模型，每条消息只会被一个 consumer 实例消费
2. 将所有的 consumer 实例划分到若干不同的 group -> 实现基于发布/订阅的模型。极端的情况是每个 consumer 实例都属于不同的 group，这样 Kafka 消息会被广播到所有 consume 实例上。

 Kafka 目前只提供单个分区内的消息顺序，而不会维护全局的消息顺序，因此如果用户要实现 topic 全局的消息读取顺序，就只能通过让每个 consumer group 下只包含一个consumer 实例的方式来间接实现(或者也可以基于业务逻辑自己控制实现)。

独立消费者只有一个消费者实例单独执行消费操作。

### 构建 consumer
构造一个 consumer 需要以下6个步骤：
1. 构造一个 `Properties` 对象，至少指定 `bootstrap.servers` 、 `key.serializer` 和 `value.serializer` 和 `group.id` 的值。
2. 使用上一步创建的 `Properties` 实例构造 `KafkaConsumer` 对象
3. 调用 `KafkaConsumer.subscribe` 方法订阅感兴趣的 topic 列表。
4. 循环调用 `KafkaConsumer.pull` 方法获取封装为 `ConsumerRecord` 的 topic 信息。
5. 处理获取到的 `ConsumerRecord` 对象。
6. 关闭 `KafkaConsumer` 。

```
class Consumer implements Runnable{
    @Override
    public void run() {
        Properties consumerProp = new Properties();
        consumerProp.put("bootstrap.servers","[::1]:9093");
        consumerProp.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
        consumerProp.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
        consumerProp.put("group.id","test-group"+ Math.random());
        consumerProp.put("auto.offset.reset","earliest");
        KafkaConsumer<String,String> consumer = new KafkaConsumer<>(consumerProp);
        List topics = new ArrayList<String>();
        topics.add("quick-start");
        consumer.subscribe(topics);
        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(100);
                for (ConsumerRecord record : records) {
                    System.out.println(record.key() + ":" + record.value());
                }
            }
        }finally {
            consumer.close();
        }
    }
}
```

### Consumer 的主要参数
1. `session.timeout.ms`
   Kafka的消费者协调者组件检测消费端（消费端宕机）失败的时间。
2. `max.poll.interval.ms`
   用于设置消息处理逻辑的最大时间。
3. `auto.commit.interval.ms`
   设置自动提交位移间隔时间，默认为 5 秒。
4. `auto.offset.reset`
   指定了无位移信息或位移越界（即consumer要消费的消息的位移不在当前消息日志的合理区间范围）时Kafka的应对策略。目前参数有三个值：
    + earliest：从最早的位移开始消费。不一定是0。
    + latest: 从最新位移处开始消费。
    + none
5. `enable.auto.commit`
   该参数指定 consumer 是否自动提交位移。如果设置为 false ，则需要用户在程序中手动提交位移。
6. `fetch.max.bytes`
   指定了 consumer 端单次获取数据的最大字节数。
7. `max.poll.records`
   该参数控制单次 poll 调用返回的最大消息数。
8. `heartbeat.interval.ms`
   该参数和
9.  `connections.max.idle.ms`
   Kafka 会定期地关闭空闲 socket 连接，这个参数指定空闲时间。如果超过空闲时间，那么 socket 连接会被关闭，下次要处理请求时，需要重新创建连接 broker 的 socket 连接。

### 重平衡
consumer group 的 rebalance 本质上是一组协议，它规定了一个 consumer group 是如何达成一致来分配订阅 topic 的所有分区的 。 

**topic 的每个分区只会分配给每个组内的一个 consumer 实例。**

Kafka内置的一个全新的组协调协议 (group coordination protocol) 负责重平衡 。对于每个组而言， Kafka 的某个broker 会被选举为组协调者（ group coordinator). coordinator 负责对组的状态进行管理，它的主要职责就是当新成员到达时促成组内所有成员达成新的分区分配方案，即 coordinator 负责对组执行 rebalance 操作。

组 rebalance 触发的条件有以下 3 个：
+ 组成员发生变更，比如新 consumer 加入组，或己有 consumer 主动离开组，再或是己有 consumer 崩溃时则触发 rebalance 。
+ 组订阅 topic 数发生变更，比如使用基于正则表达式的订阅，当匹配正则表达式的新topic 被创建时则会触发 rebalance，因为这会导致消费的分区数发生变化 。
+ 组订阅 topic 的分区数发生变更，比如使用命令行脚本增加了订阅 topic 的分区数 。

分区分配策略有3种，分别是：
1. range 策略，默认策略，可以使用`partition.assignment.strategy`参数修改
2. round-robin 策略
3. sticky 策略
   
range 策略主要是基于范围的思想。它将单个 topic 的所有分区按照顺序排列，然后把这些分区划分成固定大小的分区段井依次分配给每个 consumer; round-robin 策略则会把所有 topic 的所有分区顺序摆开，然后轮询式地分配给各个 consumer 。 最新发布的 sticky 策略有效地避免了上述两种策略完全无视历史分配方案的缺陷，采用了“有黏性”的策略对所有 consumer 实例进行分配，可以规避极端情况下的数据倾斜并且在两次 rebalance I司最大限度地维持了之前的分配方案 。

另外 Kafka 支持自定义的分配策略，用户可以创建自己的 consumer 分配器（ assignor ） 。