# consumer 原理
{docsify-updated}

1. 连接 Kafka 集群
	+ 消费者通过配置 Kafka broker 地址（bootstrap.servers）连接到 Kafka 集群。
	+ Kafka 会返回集群元数据（主题、分区信息等）。



2. 加入消费者组（Consumer Group）
	+ 每个 Kafka 消费者属于一个 Consumer Group（通过 group.id 指定）。
	+ Kafka 以组为单位为消费者分配分区，实现水平扩展。
	+ 多个消费者在同一组时，每个分区只会分配给组内的某一个消费者。
```
Kafka Cluster
 └── Topic: "orders"
      ├── Partition 0  ──► Consumer A
      ├── Partition 1  ──► Consumer B
      └── Partition 2  ──► Consumer A

Consumer Group: "order-processor"
```

3. 分区分配（Partition Assignment）
	+ Kafka 通过**协调者（Group Coordinator）**为每个消费者分配特定分区。
	+ 分区分配策略有多种（Range, RoundRobin, Sticky, Custom 等）。



4. 拉取消息（Polling）
	+ 消费者通过定期调用 poll() 方法，从所分配的分区中拉取消息。
	+ Kafka 是 pull 模型（消费者主动拉取），不是 push。
	+ poll() 会阻塞一定时间等待数据或空闲。



5. 处理消息
	+ 消费者从 poll() 返回的消息集合中处理每条消息。
	+ 你可以对消息进行业务逻辑处理、入库、写文件等。


6. 提交 offset（位移）
    Kafka 通过 offset 来记录消费进度：
    + 自动提交（enable.auto.commit=true）:Kafka 会定时提交最近 poll 到的 offset（默认 5 秒）。
    + 手动提交:可以通过 commitSync() 或 commitAsync() 手动提交。

    Offset 的提交代表“我已经成功消费并处理完这条消息”。

7. 心跳机制 & 再均衡（Rebalance）
	+ Kafka 定期通过心跳机制确认消费者存活。
	+ 如果某个消费者宕机或长时间不 poll，Kafka 会触发 Rebalance，将分区重新分配给其他存活的消费者。
	+ 在 Rebalance 过程中，消费会短暂中断，确保每个分区只被一个消费者处理。