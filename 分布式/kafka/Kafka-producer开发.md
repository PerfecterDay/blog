# Kafka-producer 开发 
{docsify-updated}
>https://www.instaclustr.com/blog/a-visual-understanding-to-ensuring-your-kafka-data-is-literally-in-order/

### Java版 producer 工作流程
如下图所示：
<center><img src="pics/kafka-producer.png" alt="" width="50%"></center>

producer 首先使用一个线程将待发送的消息封装进一个 `ProducerRecord` 类实例，然后将其序列化之后发送给 partitioner，再由后者确定了目标分区后一同发送到位于 producer 程序内部的一块内存缓冲区中。 producer 的另一个 IO 线程则负责实时地从该缓冲区中提取出准备就绪的消息封装进一个批次，统一发送给对应的 broker 。

构造一个 Kafka producer 实例大致需要5个步骤：
1. 构造一个 `Properties` 对象，至少指定 `bootstrap.servers` 、 `key.serializer` 和 `value.serializer` 这3个属性。或者这些参数可以在构造 `KafkaProducer` 时传人构造函数。这样 `Properties` 就不用指定这些参数了。
2. 使用上一步创建的 `Properties` 实例构造 `KafkaProducer` 对象
3. 构造要发送的消息对象 `ProdecerRecord`，指定消息要被发送到的 topic、分区以及对应的 key 和 value 。注意，分区和 key 信息可以不用指定，由 Kafka 自行确定目标分区。
4. 调用 `KafkaProducer` 的 send 方法发送消息。
5. 关闭 KafkaProducer 。

Demo:
```
Properties kafkaProp = new Properties();
kafkaProp.put("bootstrap.servers","172.26.238.80:9092");
kafkaProp.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
kafkaProp.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
Producer<String,String> producer = new KafkaProducer<String, String>(kafkaProp);
for (int i = 0; i < 10; i++) {
   producer.send(new ProducerRecord<>("quick-start",Integer.toString(i),Integer.toString(i)));
}
```

#### producer 的主要参数
1. `acks`  
   这个参数制定了kafka服务端在给 `producer` 发送响应前， lead broker 端必须要确保已经写入该消息的副本的数量。当前有3个值：0、1和 all。
   + `acks=0`：表示 producer 完全不理睬 leader broker 端的处理结果。吞吐量最高，但是消息可能会写入失败。
   + `acks=all` 或者 `-1`: leader broker 不仅会将消息写入本地日志，同时还会等待 ISR 中的所有其他副本都成功写入它们各自的本地日志后才返回相应结果给 `producer` 端。这样可以保证当 ISR 中至少有一个副本是存活状态时，消息是不会丢失的。但是吞吐量也是最低的。
   + `acks=1`：是0和all的折中方案，也是默认值。leader broker 收到消息后将消息写入本地日志，然后就发送相应结果给 producer ，无需等待 ISR 中其它副本写入该消息。
    <center><img src="pics/kafka-acks.png" alt="" width="60%"></center>
2. `buffer.memory`
   该参数指定了 `producer` 端用于缓存消息的缓冲区的大小，单位是字节，默认值是 32MB。如前所述， `producer` 主线程会把要发送的消息放入缓冲区，然后由IO线程发送这些消息。
3. `compression.type`
   设置 `producer` 端是否压缩消息，默认不压缩。主要的压缩算法有： GZIP、Snappy、LZ4、Zstandard。
4. `retries`
   当消息发送失败时，该参数能指定重试的次数。值得注意的是重试可能造成消息重复发送和消息的乱序。
5. `batch.size`
   前面提到过， `producer` 会把发往同一分区的多条消息封装进一个 batch 中。当 batch 满了的时候， `producer` 会发送 batch 中的所有消息。不过， `producer` 并不总是等待 batch 满了才发送消息。 `batch.size` 默认 16KB。
7. `max.block.ms`
   Producer 在调用 `send()` 或相关方法时，最多允许被**阻塞多久**
   ```
   send() 卡住，通常是因为本地缓冲区（buffer.memory）满，或者获取元数据超时 → 超过 max.block.ms → 直接抛异常
   ```
8. `linger.ms`
   该参数控制的是消息发送的延时，默认值是0，表示消息需要被立即发送，无需关心 batch 是否已被填满。
9.  `request.timeout.ms`
   设置的是单次请求等 broker 响应的超时时间，如果发送后，在该参数指定的时间内， `producer` 没有收到 broker 的响应，那么 `producer` 就认为该请求超时了。
10. `delivery.timeout.ms`
   从 `send()` 返回开始计时，到结果出来为止的最大等待时间。它控制的是“总投递时间”，不是单次请求。
11. `parttioner.class`
   设置自定义的 parttioner 分区机制。
12. `enable.idempotence`
   幂等性设置，启用以后，相同的消息（错误重试的消息）只会被集群存储一次。
13. `max.in.flight.requests.per.connection`：每个 TCP 连接上，最多允许 5 个未确认请求同时发送。若开启幂等性，最大只能是 5。
14. `max.request.size`
    一次网络请求能发多大的数据包，防止 Producer 把超大请求砸给 broker。它限制的不是“单条消息”，而是“单个请求”.
    ```
    多条消息 → 组成 batch
    多个 batch → 组成一个 ProduceRequest
    ProduceRequest 大小 ≤ max.request.size
    ```

```
send() （max.block.ms）
  ↓
进入 batch 等待 linger.ms
  ↓
多次重试 (request.timeout.ms * retries)
  ↓
只要总时间 > delivery.timeout.ms
      → 直接失败
```

####  ProducerRecord
`ProducerRecord` 代表了一条要发送的消息，由5个字段构成，它们分别如下:
+ `String topic` ：该消息所属的 topic 
+ `Integer partition` ：直接指定该消息所属的分区 
+ `Headers headers`: 消息头
+ `K key` ：消息 key 
+ `V value` ：消息体
+ `Long timestamp` ：消息时间戳，慎重使用这个功能，因为它有可能会令时间戳索引机制失效

#### RecordMetadata
该数据结构表示 Kafka 服务器端返回给客户端的消息的元数据信息，包含如下内容 。
+ `long offset` ：消息在分区日志中的位移信息。
+ `long timestamp` ：消息时间戳 。
+ `TopicPartition topicPartition` ：所属 topic 及分区 。
+ `int serializedKeySize` ：序列化后的消息 key 字节数。
+ `int serializedValueSize` ：序列化后的消息 value 字节数 。

#### 消息分区机制
Kafka 的消息发送过程中，很重要的一步就是要确定将消息发送到指定的 topic 的哪个分 区中。默认的分区策略会尽力确保具有相同 key 的所有消息都会被发送到相同的分区上；如果没有为消息指定 key ，则会选择轮询的方式来确保消息在 topic 的所有分上均匀分布。

 producer 提供了分区策略以及对应的分区器 (`Partitioner`) 供用户使用 如果用户需要自定义分区机制，就需要完成两件事：
1. 在 producer 程序中创建一个类，实现 `org.apache.kafka.clients.producer.Partitioner` 接口，主要分区逻辑在 `partition()`方法中实现。
2. 在用于构造 `KafkaProducer`的 Properties 对象中设置 `parttioner.class`参数。

#### 保证顺序
Kafka 只能保证单个 partion 内的消息是有序的。如果发送到同一个 topic 的消息被发送到多个 partion ，那么在消费时就不能保证顺序。注意
[消息顺序](https://stackoverflow.com/questions/29511521/is-key-required-as-part-of-sending-messages-to-kafka)

### 消息序列化
在网络中发送数据都是以**字节**的方式传输， Kafka 也不例外。 Apache Kafka 支持用户给 broker发送各种类型的消息。它可以是一个字符串、一个整数、一个数组或是其他任意的对象类型。
序列化器（serializer）负责在 producer 发送前将消息转换成字节数组；而与之相反，解序列化器（deserializer）则用于将 consumer 接收到的字节数组转换成相应的对象。
<center><img src="pics/kafka-serializer.png" width="60%"></center>

序列化器负责在 producer 发送前将消息转换成字节数组；解序列化器则用于在 consumer 接收到的字节数组转换成相应的对象。Kafka 默认提供了十几种序列化器，常用的如下：
+ ByteArraySerializer: 序列化字节数组。
+ ByteBufferSerializer: 序列化 ByteBuffer。
+ ByteSerializer: 序列化 Kafka 自定义的 Bytes 类。
+ DoubleSerializer: 序列化 Double 类型。
+ IntegerSerializer：序列化 Integer 类型。
+ LongSerializer: 序列化 Long 类型。
+ StringSerializer: 序列化 String 类型。

若涉及到复杂类型的序列化，则需要用户自定义消息序列化器。编写一个自定义的 Serializer ，需要以下几步：
1. 定义数据对象格式。
2. 创建自定义的序列化类，实现 `org.apache.kafka.common.serialization.Serializer` 接口，在 `serializer` 方法中实现序列化逻辑。
3. 在用于构造 KafkaProducer 的 Properties 对象中设置 `key.serializer` 或 `value.serializer` :取决于是为消息的 key 还是 value 做自定义序列化。

producer 的序列化机制使用起来非常简单，只需要在构造 producer 时同时指定 参数 `key.serializer` 和 `value.serializer` 的值即可。
```
props.put ("key.serializer","org.apache.kafka.common.serialization.IntegerSerializer"); 
props.put ("value.serializer","org.apache.kafka.common.serialization.StringSerializer"); 
```
用户可以为消息的 key 和 value 指定不同类型的 serializer ，只要与解序列类型分别保持一致就可以。

### Producer 拦截器
对于 producer 而言，消息拦截器（interceptor）使得用户在消息发送之前以及收到 broker 响应执行 producer 回调逻辑之前有机会对消息做一些定制化的需求。同时允许指定多个 `interceptor` 按序作用于同一条消息从而形成一个拦截链。主要就是实现 `org.apache.kafka.clients.producer.ProducerInterceptor` 接口。


tips:
在 windows wsl 环境中启动 kafka 集群时，如果要在宿主机连接到该集群，必须在 kafka 配置 listener 时指定 ip 地址，在客户端连接时，也用 ip 地址连接。否则会有网络问题。

### 无消息丢失配置
Java 版本 producer 用户采用异步发送机制。 KafkaProducer.send 方法仅仅把消息放入缓冲区中，由一个专属 1/0 线程负责从缓冲区中提取消息井封装进消息 batch 中，然后发送出去。显然，这个过程中存在着数据丢失的窗口：若 IO 线程发送之前 producer 崩溃，则存储缓冲区中的消息全部丢失了。

producer 的另一个问题就是消息的乱序。假设客户端依次执行下面的语句发送两条消息到相同的分区：
```
producer.send(record1); 
producer.send(record2);
```
若此时由于某些原因（比如瞬时的网络抖动〉导致 record I 未发送成功，同时 Kafka 又配置了**重试机制**以及 `max.in.flight.requests.per.connection` 大于 1 （默认值是 5 ），那么 producer 重试 record1 成功后， record1 在日志中的位置反而位于 record2 之后，这样造成了消息的乱序。

无消息丢失的 producer 配置列表：
+ ` max.block.ms=500`: 使得内存缓冲区被填满时 producer 处于阻塞状态的最长时间
+ `acks=all or 1`：必须要等到所有 fo llower 都响应了发送消息才能认为提交成功，这是 producer 端最强程度的持久化保证。
+ `retries = Integer.MAX_VALUE`：是 producer 要开启无限重试
+ `max.in.flight.requests.per.connection= 1`：设置该参数为 1 主要是为工防止 topic 同分区下的消息乱序问题。这个参数的实际效果其实限制了 producer 在单个 broker 连接上能够发送的未响应请求的数量。因此，如果设置成 l , 则 producer 在某个 broker 发送响应之前将无法再给该 broker 发送 PRODUCE 请求 。
+ 使用带回调机制的 send 发送消息，即 `KafkaProducer.send(record , callback)`
+ Callback 逻辑中显式地立即关闭 producer ，使用 close(0)
+ `enable.auto.commit= false` : 关闭自动提交

无消息丢失的 broker 配置列表：
+ `unclean.leader.election.enable=false`:即不允许非 ISR 中的副本被选举为 leader ，从而避免 broker 端因日志水位截断而造成的消息丢失 。
+ `replication.factor = 3`: 多个副本保存消息
+ `min.insync.replicas > 1 `: 用于控制某条消息至少被写入到 ISR 中的多少个副本才算成功，设置成大于 1 是为了提升producer 端发送语义的持久性
+ `replication.factor > min.insync.replicas` : 若两者相等，那么只要有一个副本挂掉，分区就无法正常工作，虽然有很高 的持久性但可用性被极大地降低了

### 多线程处理
实际环境中只使用一个用户主线程通常无法满足所需的吞吐量目标，因此需要构造多个线程或多个进程来同时给 Kafka 集群发送消息。这样在使用过程中就存在着两种基本的使用方法。
+ 多线程单 KatkaProducer 实例 ：这种方法就是在全局构造一个 KatkaProducer 实例，然后在多个线程中共享使用。由于 KatkaProducer 是线程安全的，所以这种使用方式也是线程安全的。
+ 多线程多 KatkaProducer 实例 ：每个 producer 线程中都单独构造一个 KafkaProducer 实例，并保证此实例在该线程中封闭。

笔者比较推荐使用第 一 种方法，即在多个producer 用户线程中共享一个 KafkaProducer 实例 。若是对那些拥有超多分区的集群而言，采用第二种方法具有较高的可控性，方便 producer 的后续管理。