# Spring JMS集成
{docsify-updated}

### JMS 简介
在JMS出世之前，存在各种MOM（Message-Oriented Middleware)产品，不同产品使用自己的API，所以企业信息系统在使用这些MOM时，不得不使用各个MOM特定的API，这些API可能基于不同的语言实现，为了能够在JAVA平台上使用消息机制开发的程序有一个统一的访问不同MOM产品的方式，于是诞生了JMS规范。JMS规范是一套API规范，类似于JDBC。使得在访问消息中间件时，能够使用一套统一的API规范，这样即使切换了中间件系统，也不用重构应用代码。

JMS定义了两种消息模型：
1. 点对点模式。消息发者将消息发送到指定的消息队列，消息接收者也是从同一个消息队列中顺序获取数据，并对获取的消息进行处理。这种模式类似于生产者-消费者模式
2. 发布订阅模式。消息发送者将消息发送到指定的消息主题（topic），多个消息接收者可以从相应的主题中获取某个相同消息的拷贝并进行处理。这种模式类似于报纸杂志的订阅场景。

一个典型的JMS应用通常由以下几个部分组成：
+ JMS类型的客户端程序。使用Java开发的用于发送或者接收消息的程序。
+ 非JMS类型的客户端程序。使用消息系统提供的特定API开发的客户端程序，在JMS之前，这种客户端是比较常见的。
+ 消息。JMS定义了5种消息类型： StreamMessage 、 ByteMessgae 、 MapMessage 、 TextMessage 以及 ObjectMessage。
+ JMS Provider 。JMS规定了一套API接口规范，加入JMS规范的各种MOM产品，需要根据规范提供特定于自身产品的JMS实现，这些特定MOM产品的JMS API 实现及相关功能实现称之为 JMS Provider 。
+ 受管理对象。为了向不同JMS类型客户端程序屏蔽各种 JMS Provider 之间的差异性，系统管理员可以将某些特定类型的对象，通过 JMS Provider 提供的管理工具提前配置到系统中。这些提前配置到系统然后再交给JMS客户端使用的对象就叫做受管理对象。有如下两种类型：
    1. ConnectionFactory ：客户端通过 ConnectionFactory 获取 Connection 之后才能访问消息系统。
    2. Destination ：即消息发送或者接收的目的地，就是配置好的 Queue 和 Topic 。

### 使用JMS API开发的传统套路
使用JMS API发送消息的一般过程：
1. 首先，从 JNDI 获取 JMS 的受管理对象，即 ConnectionFactory 和稍后要用到的一个或多个 Destination 引用。
2. 使用获取 ConnectionFactory 创建发送消息的 Connection
3. 使用 Connection 创建发送消息的 Session
4. 使用 Session 创建发送消息的 MessageProducer
5. 使用 Session 创建将被发送的消息
6. 使用 MessageProducer 发送的消息到指定的 Destination
7. 最后做资源管理，包括 MessageProducer、Session以及Connection的关闭

### kafka的Docker镜像使用说明(wurstmeister/kafka)
0. 编写一个 docker-compose.yml 文件：
```
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://:9092
      KAFKA_LISTENERS: PLAINTEXT://:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
1. 启动 Server: `docker-compose up -d`
2. 扩展 broker: `docker-compose up -d --scale kafka=4`
3. 创建 topic: `docker exec kafka_kafka_1 kafka-topics.sh --create --topic topic001 --partitions 4 --zookeeper zookeeper:2181 --replication-factor 2`
4. 查看 topic: `docker exec kafka_kafka_3 kafka-topics.sh --list --zookeeper zookeeper:2181`
5. 查看 topic 详细信息： `docker exec kafka_kafka_3 kafka-topics.sh --describe --topic topic001 --zookeeper zookeeper:2181`
6. 消费消息： `docker exec kafka_kafka_2 kafka-console-consumer.sh --topic topic001 --bootstrap-server kafka_kafka_1:9092,kafka_kafka_2:9092,kafka_kafka_3:9092,kafka_kafka_4:9092`
7. 生产消息： `docker exec -it kafka_kafka_1 kafka-console-producer.sh --topic topic001 --broker-list kafka_kafka_1:9092,kafka_kafka_2:9092,kafka_kafka_3:9092,kafka_kafka_4:9092`
