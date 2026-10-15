# 消息交付语义
{docsify-updated}

kafka支持3种消息交付语义：
+ 最多一次（ at most once ）处理语义：消息可能丢失，但不会被重复处理 。
+ 最少一次（ at least once ）处理语义：消息不会丢失，但可能被处理多次。
+ 精确一次（ exactly once ）处理语义：消息一定会被处理且只会被处理一次 。

## 幂等性
对 producer 而言， Kafka 引入己提交消息 committed message ）的概念 。一旦消息被成功地提交到日志文件，只要至少存在一个可用的包含该消息的副本，那么这条消息就永远不会丢失。在 0.11.0 .0 版本之前， Kafka producer 默认提供的是 at least once 语义 。

设想这样的一个场景，当 producer 向 broker 发起新消息后，分区 leader 副本所在的 broker 成功地将该消息写入本地磁盘，然后发送响应给 producer。 此时假设网络出现故障导致该响应没有发送成功，那么未接到响应的 producer 会认为该消息请求失败从而开启重试操作。若重试后网络恢复正常，那么显然同一条消息被写入到日志两次。在比较极端的情况下，同一条消息可能会被发送多次。

0.11.0.0 版本引入了幂等性 producer 表示它的发送操作是幂等的 。 瞬时的发送错误可能导致 producer 端出现重试，同一条消息被 producer 发送多次，但在 broker 端这条消息只会被写入日志一次。 对于单个 topic 分区而言 ，这种 producer 提供的幂等性消除了各种错误导致的重复消息。 如果要启用幂等 性 producer 以及获取其提供的 EOS 语义，用户需要显式地设置 producer 端的新参数 `enable.idempotence` 为 true 。

幂等性 producer 的设计思路类似于 TCP 的工作方式，发送到 broker 端的每批消息都会被赋予一个**序列号**（ sequence number ）用于消息去重。但是和 TCP 不同的是，这个序列号不会被丢弃，相反 Kafka 会把它们保存在底层日志中，这样即使分区的 leader 副本挂掉，新选出来的 leader broker 也能执行消息去重工作。 保存序列号只需要额外几字节，因此整体上对 Kafka消息保存开销的影响并不大。除了序列号， Kafka 还会为每个 producer 实例分配一个 producer id （下称 PID ）。 producer在初始化时必须分配一个 PID 。 PID 分配的过程对用户来说是完全透明的，因此不会为用户所见。 消息要被发送到的每个分区都有对应的序列号值，它们总是从 0 开始并且严格单调增加 。对于 PID 、分区和序列号的关系，用户可以设想一个 Map, key 就是 `(PID ，分区号)`, value 就是**序列号**。即每对（PID ，分区号）都有对应的序列号值。若发送消息的序列号小于或等于broker 端保存的序列号，那么 broker 会拒绝这条消息的写入操作 。

<center>
	<img src="pics/kafka-idempotent.png" width="40%">
</center>

## 事务
> https://zhuanlan.zhihu.com/p/411171789?spm=a2c6h.13046898.publish-article.8.20506ffawEKTMZ

对事务的支持是 Kafka 实现 EOS 的第二个利器。