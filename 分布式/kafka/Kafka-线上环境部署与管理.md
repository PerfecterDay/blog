#  Kafka线上环境部署管理及监控
{docsify-updated}

## Kafka 参数配置
1. broker 端参数  
   + `broker.id` : Kafka 使用唯一的整数标识来标识集群中的每个 broker。
   + `log.dirs` : 指定了 Kafka 持久化消息的目录。可以设置多个目录，以逗号分隔。单机启动多个 kafka 实例时，需要指定不同的目录。
   + `zookeeper.connect` : zookeeper 集群的连接信息，逗号分隔。
   + `listeners` : 逗号分隔的 broker 监听器列表。格式为 协议：//主机名：端口,格式为 协议：//主机名：端口... 用于监听客户端的连接请求。
   + `delete.topic.enable` : 是否允许 Kafka 删除 topic 。
   + `log.retention.{hours|minutes|ms}`：控制消息数据的留存时间。默认保留7天
   + `log.retention.bytes` : 控制了每个消息日志保存多大数据，超过改参数的分区日志，会被自动清除。
   + `min.insync.replicas`: 与发布端（prducer）的 acks 参数配合使用。它指定了 broker 端必须在指定数量的副本中持久化成功后，才响应发布端消息发送成功。
   + `num.network.threads` : 指定 broker 端实际处理网络请求的线程数。
   + `num.io.threads` ：指定 broker 端用于处理消息的 IO 线程数
2. topic 级别参数  
   + `delet.retention.ms` : 为每个 topic 设置自己的日志过期时间，覆盖 broker 端的全局设置。
   + `retention.bytes` : 类似 `log.retention.bytes` 。
3. GC 配置参数
4. JVM 参数
5. OS 参数


```
$ /opt/homebrew/var/lib/kraft-combined-logs/cms.cap.sync-0 git:(main) 0 • +0 -0
tree
.
├── 00000000000000000010.index
├── 00000000000000000010.log
├── 00000000000000000010.snapshot
├── 00000000000000000010.timeindex
├── leader-epoch-checkpoint
└── partition.metadata

1 directory, 6 files
```

## Kafka的启动与管理
1. broker 管理  
   1. 启动ZK：`zookeeper-server-start.sh config/zookeeper.properties`
   2. 启动Kafka broker: `kafka-server-start.sh config/server.properties -daemon`
   3. 开启 JMX ：`export JMX_PORT=9999 zookeeper-server-start.sh config/zookeeper.properties`
   4. 关闭Kafka broker:  
      1. 前台启动方式的关闭：`Ctrl+C`
      2. 后台启动方式的关闭：`kafka-server-stop.sh`
2. topic 管理
   1. 创建topic: `kafka-topics.sh --create --topic quick-start --partitions 1 --bootstrap-server localhost:9092 --replication-factor 1`
   2. 查看topic信息: `kafka-topics.sh --describe --topic quick-start --bootstrap-server localhost:9092`
   3. 查看所有的 topic 列表：`kafka-topics.sh --bootstrap-server localhost:9092 --list`
   4. 删除 topic : `kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic quick-start`
   5. 修改 topic : `kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic quick-start --partions 10`
   6. kafka-topics 脚本命令行参数：
   <center><img src="pics/kafka-topics.png" alt="" width="60%"></center>
3. 写消息到 topic  
    ```
    kafka-console-producer.sh --topic quick-start --bootstrap-server localhost:9092
    
    //发送消息时指定 key
    kafka-console-producer --topic cms.cap.sync --bootstrap-server localhost:9092 --property "parse.key=true" --property "key.separator=:"

    > mykey:{"auditId":601339,"tableName":"ClntAccSecMain","operation":"UPDATE","recordId":"800888","timestamp":"2026-07-02T15:53:19.067","values":{"AccountCode":"800888","StatusIndex_N":"Closed","AccountType":"CASH","InterenetTrade":"Yes"}}
    ```

4. 从指定topic读取消息  
   `kafka-console-consumer.sh --topic quick-start --from-beginning --bootstrap-server localhost:9092`  
   `kafka-console-consumer --topic topic2 --from-beginning --group=test --bootstrap-server localhost:9092`

## Kafka 集群监控
1. CMAK(原名 Kafka Manager)：https://github.com/yahoo/CMAK
2. kafka-ui: https://github.com/provectus/kafka-ui?tab=readme-ov-file  
   `docker run -it --name=kafka-ui -p 9090:8080 -e DYNAMIC_CONFIG_ENABLED=true provectuslabs/kafka-ui`


## Kafak 负载
1. 客户端先连 bootstrap server / F5 VIP kafka.demo.net:9094
2. 连接成功后，broker 会返回 集群 metadata
3. metadata 里带的是每个 broker 的 advertised.listeners 地址
4. 客户端接下来会绕过 F5，直接去连这些 broker
5. 所以如果这 3 个 broker 地址对客户端网络不通，最终还是不能正常收发消息

Kafka 的 `bootstrap.servers` 只是入口，不是全程代理入口。它的作用只是：帮客户端先找到集群，拿到分区 leader / broker metadata。  
后续真正的数据读写，是客户端直接和对应 broker 通信。所以 VIP 不是 HTTP 反向代理那种“只通一个入口就够了”。