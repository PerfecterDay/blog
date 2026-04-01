#  Kafka 中的位移
{docsify-updated}

- [Kafka 中的位移](#kafka-中的位移)
  - [位移](#位移)
  - [`__consumer_offsets`](#__consumer_offsets)
    - [查看 `__consumer_offsets` 的数据格式](#查看-__consumer_offsets-的数据格式)
    - [检查各个消费者情况](#检查各个消费者情况)


### 位移
Offset 记录了 consumer 实例消费的消息的位置，Kafka 让 consumer 端保存 offset 。consumer 客户端需要定期地向 Kafka 集群汇报自己消费数据的进度，这一过程被称为**位移提交**。Kafka 内部有一个专门用来记录 consumer 端位移信息的 topic —— `__consumer_offsets` 。consumer 端需要为每个它要读取的分区保存消费进度，即分区中当前最新消费消息的位置 。该位置就被称为位移（ offset ） 。 consumer 需要定期地向 Kafka 提交自己的位置信息，实际上，这里的位移值通常是下一条待消费的消息的位置。假设 consumer 己经读取了某个分区中的第N条消息，那么它应该提交位移值为 N，因为位移是从 0 开始的，位移为 N 的消息是第 N+l 条消息 。 这样下次 consumer 重启时会从第 N+l 条消息开始消费。总而言之， offset 就是 consumer 端维护的位置信息 。

offset 对于 consumer 非常重要，因为它是实现消息交付语义保证（ message delivery semantic ）的基石 。 常见的 3 种消息交付语义保证如下。
+ 最多一次（ at most once ）处理语义：消息可能丢失，但不会被重复处理 。
+ 最少一次（ at least once ）处理语义：消息不会丢失，但可能被处理多次。
+ 精确一次（ exactly once ）处理语义：消息一定会被处理且只会被处理一次 。

consumer 提交位移的主要机制是通过向所属的 coordinator 发送位移提交请求来实现的 。每个位移提交请求都会往`＿consumer_offsets` 对应分区上追加写入一条消息 。 消息的 key 是group.id 、 topic 和分区的元组，而 value 就是位移值。 如果 consumer 为同一个 group 的同一个topic 分区提交了多次位移，那么一`consumer_offsets` 对应的分区上就会有若干条 key 相同但value 不同的消息，但显然我们只关心最新一次提交的那条消息。从某种程度来说，只有最新提交的位移值是有效的，其他消息包含的位移值其实都已经过期了 。 Kafka 通过压实（ compact)策略来处理这种消息使用模式。

默认情况下， consumer 是自动提交位移的，自动提交间隔是 5 秒。这就是说若不做特定的设置 ， consumer 程序在后台自动提交位移。通过设置 `auto.commit.interval.ms` 参数可以控制自动提交的间隔。

自动位移提交的优势是降低了用户的开发成本使得用户不必亲自处理位移提交：劣势是用户不能细粒度地处理位移的提交，特别是在有较强的精确一次处理语义时。在这种情况下，用户可以使用手动位移提交。

设置使用手动提交位移非常简单，仅仅需要在构建 KafkaConsumer 时设置 enable.auto.comrnit=false ，然后调用 comrnitSync 或commitAsync 方法即可。

### `__consumer_offsets`
位移记录的是以 `(group_id, topic, partion, offset)` 元组的形式保存的，可以把它想象成一个 KV 格式的消息， key就是一个三元组: `(group.id,topic,分区号)`， 而 value就是 offset 的值。 每当更新同一个 key的最新 offset值时，该 topic就会写入一条含有 最新 offset 的消息，同时 Kafka会定期地对该 topic执行压实操作(compact)，即为每个消息 key 只保存含有最新 offset 的消息。 这样既避免了对分区日志消息的修改 ，也 控制住了 __consumer_offsets topic总体的日志容量 ，同时还能实时反映最新的消费进度。

<center><img src="pics/consumer_offset.jpg" width="50%"></center>

#### 查看 `__consumer_offsets` 的数据格式
`kafka-console-consumer --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server localhost:9092 --topic __consumer_offsets --from-beginning`

输出如下：
```
[bj,G_P_USER_PROD_CPZX2HK_DEV,1]::OffsetAndMetadata(offset=0, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166373455, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,0]::OffsetAndMetadata(offset=0, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166373455, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,2]::OffsetAndMetadata(offset=0, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166373455, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,2]::OffsetAndMetadata(offset=1, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166471643, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,2]::OffsetAndMetadata(offset=2, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166582269, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,2]::OffsetAndMetadata(offset=3, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166707883, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,2]::OffsetAndMetadata(offset=4, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166719632, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,1]::OffsetAndMetadata(offset=0, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166804585, expireTimestamp=None)
[bj,G_P_USER_PROD_CPZX2HK_DEV,0]::OffsetAndMetadata(offset=0, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706166804585, expireTimestamp=None)
```

`[bj,G_P_USER_PROD_CPZX2HK_DEV,0]::OffsetAndMetadata(offset=18, leaderEpoch=Optional.empty, metadata=, commitTimestamp=1706174258923, expireTimestamp=None)`
代表 group_id = bj 的 consumers 在 topic=G_P_USER_PROD_CPZX2HK_DEV 的 partion=0 的 partion 上的最新的消费到的位移位置是 18.

#### 检查各个消费者情况
`kafka-run-class.sh kafka.admin.ConsumerGroupCommand --bootstrap-server localhost:9092 --group bj --describe`