#  Kafka线上环境部署管理及监控
{docsify-updated}

- [Kafka线上环境部署管理及监控](#kafka线上环境部署管理及监控)
    - [Kafka 参数配置](#kafka-参数配置)
    - [Zookeeper 配置文件](#zookeeper-配置文件)
    - [Kafka的启动与管理](#kafka的启动与管理)
    - [Kafka 集群监控](#kafka-集群监控)


### Kafka 参数配置
1. broker 端参数  
   + `broker.id` : Kafka 使用唯一的整数标识来标识集群中的每个 broker。
   + `log.dirs` : 指定了 Kafka 持久化消息的目录。可以设置多个目录，以逗号分隔。单机启动多个 kafka 实例时，需要指定不同的目录。
   + `zookeeper.connect` : zookeeper 集群的连接信息，逗号分隔。
   + `listeners` : 逗号分隔的 broker 监听器列表。格式为 协议：//主机名：端口,格式为 协议：//主机名：端口... 用于监听客户端的连接请求。
   + `delete.topic.enable` : 是否允许 Kafka 删除 topic 。
   + `log.retention.{hours|minutes|ms}`：控制消息数据的留存时间。默认保留7天
   + `log.retention.bytes` : 控制了每个消息日志保存多大数据，超过改参数的分区日志，会被自动清除。
   + `min.insync.replicas`: 与发布端（prducer)的 acks 参数配合使用。它指定了 broker 端必须在指定数量的副本中持久化成功后，才响应发布端消息发送成功。
   + `num.network.threads` : 指定 broker 端实际处理网络请求的线程数。
   + `num.io.threads` ：指定 broker 端用于处理消息的 IO 线程数
2. topic 级别参数  
   + `delet.retention.ms` : 为每个 topic 设置自己的日志过期时间，覆盖 broker 端的全局设置。
   + `retention.bytes` : 类似 `log.retention.bytes` 。
3. GC 配置参数
4. JVM 参数
5. OS 参数

### Zookeeper 配置文件
```
tickTime=2000
dataDir=/usr/zookeeper/data_dir
clientPort=2181
clientPortAddress=0.0.0.0
initLimit=5
syncLimit=2
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

+ `tickTime`: Zookeeper最小时间单位，用于丈量心跳时间和超时时间，单位毫秒。
+ `dataDir`: Zookeeper 会在内存中保存系统快照，并定期写入该路径指定的文件夹。
+ `clientPort`: Zookeeper监听客户端连接的端口。
+ `initLimit`: 指定 follower 结点初始时连接 leader 结点的最大 tick 次数，tick 时间就是 tickTime 配置的时间。
+ `syncLimit`: 设置 follower 节点与 leader 节点进行同步的最大时间。
+ `server.X=host:port1:port2` : X 是全局唯一的 ID ，与 myid 文件中的数字一致，后边的 port1 用来使 follower 节点连接 leader 节点， port2 用于 leader 选举。

搭建Zookeeper 集群需要在每个 zookeeper 服务的配置文件中的 dataDir 目录下新建一个 myid 文本文件，在文件中仅仅是一个数字，代表zookeeper 实例的ID。在单机上启动多个 zk 实例时，需要复制多份配置文件，然后在配置文件中需要为每个 server 指定不同的端口，否则会产生端口冲突。`dataDir`也要指定到不同的目录。

### Kafka的启动与管理
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
    `kafka-console-producer.sh --topic quick-start --bootstrap-server localhost:9092`
4. 从指定topic读取消息  
   `kafka-console-consumer.sh --topic quick-start --from-beginning --bootstrap-server localhost:9092`  
   `kafka-console-consumer --topic topic2 --from-beginning --group=test --bootstrap-server localhost:9092`

### Kafka 集群监控
1. CMAK(原名 Kafka Manager)：https://github.com/yahoo/CMAK
2. kafka-ui: https://github.com/provectus/kafka-ui?tab=readme-ov-file  
   `docker run -it --name=kafka-ui -p 9090:8080 -e DYNAMIC_CONFIG_ENABLED=true provectuslabs/kafka-ui`