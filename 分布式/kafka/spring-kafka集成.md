#  Kafka 与 Spring/Spring-boot 的集成
{docsify-updated}
> https://www.baeldung.com/spring-kafka  
> https://docs.spring.io/spring-boot/docs/current/reference/html/messaging.html#messaging.kafka

- [Kafka 与 Spring/Spring-boot 的集成](#kafka-与-springspring-boot-的集成)
  - [添加 maven 依赖](#添加-maven-依赖)
  - [创建 Topic](#创建-topic)
  - [生产者-发布消息](#生产者-发布消息)
  - [消费者-消费消息](#消费者-消费消息)
    - [消费特定partition的信息](#消费特定partition的信息)
    - [添加消息过滤器](#添加消息过滤器)
  - [Springboot 配置](#springboot-配置)


### 添加 maven 依赖
```
//spring
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
  <version>3.0.7</version>
</dependency>

//springboot
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

### 创建 Topic
之前，我们运行命令行工具来创建Kafka中的 topic:
```
$ bin/kafka-topics.sh --create \
  --zookeeper localhost:2181 \
  --replication-factor 1 --partitions 1 \
  --topic mytopic
```

但随着Kafka中AdminClient的引入，我们现在可以以编程方式创建Topic，在应用上下文中定义了一个KafkaAdmin Bean，它可以自动向代理添加主题。要做到这一点，你可以为应用上下文的每个主题添加一个NewTopic @Bean。2.3版本引入了一个新的类`TopicBuilder`，使创建此类Bean更加方便。下面的例子展示了如何做到这一点：我们需要添加 KafkaAdmin Spring Bean，它将自动为所有 NewTopic 类型的 bean 添加 Topic:
```
@Configuration
public class KafkaTopicConfig {
    
    @Value(value = "${spring.kafka.bootstrap-servers}")
    private String bootstrapAddress;

    @Bean
    public KafkaAdmin kafkaAdmin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
        return new KafkaAdmin(configs);
    }

// 使用 Springboot 时，直接申明 topic 即可创建 topic
    @Bean
    public NewTopic topic1() {
         return new NewTopic("baeldung", 1, (short) 1);
    }

// 直接使用 TopicBuilder 创建topic，不需要定义 `KafkaAdmin` bean
	@Bean
	public NewTopic topic2() {
		return TopicBuilder.name("thing2")
				.partitions(10)
				.replicas(3)
				.config(TopicConfig.COMPRESSION_TYPE_CONFIG, "zstd")
				.build();
	}
}
```

### 生产者-发布消息
为了创建消息，我们首先需要配置一个 `ProducerFactory` 。可以在设置创建Kafka `Producer` 实例的策略。  
然后我们需要一个 `KafkaTemplate` ，它包装了一个生产者实例，并提供了向Kafka topic 发送消息的便利方法。  
`Producer` 实例是线程安全的。因此，在整个应用环境中使用一个实例会有更高的性能。因此， `KakfaTemplate` 实例也是线程安全的，建议使用一个实例。

```
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(
          ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
          bootstrapAddress);
        configProps.put(
          ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
          StringSerializer.class);
        configProps.put(
          ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
          StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

发布消息：
```
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;

public void sendMessage(String msg) {
    kafkaTemplate.send(topicName, msg);
}
```

`send` API返回一个 `CompletableFuture` 对象。如果我们想阻止发送线程，并获得关于发送消息的结果，我们可以调用`CompletableFuture`对象的`get` API。该调用将阻塞线程等待结果返回，这会减慢生产者的速度。  
Kafka是一个快速流处理平台。因此，最好是异步处理结果，这样后续的消息就不会等待前一个消息的结果了。  
我们可以通过回调来做到这一点：
```
public void sendMessage(String message) {
     CompletableFuture<SendResult<String, String>> future = kafkaTemplate.send(topicName, message);
     future.whenComplete((result, ex) -> {
         if (ex == null) {
             System.out.println("Sent message=[" + message + 
                 "] with offset=[" + result.getRecordMetadata().offset() + "]");
         } else {
             System.out.println("Unable to send message=[" + 
                 message + "] due to : " + ex.getMessage());
         }
     });
}
```

### 消费者-消费消息
为了消费消息，我们需要配置一个 `ConsumerFactory` 和一个 `KafkaListenerContainerFactory` 。一旦这些 bean 在Spring bean 工厂中可用，就可以使用 `@KafkaListener` 注解来配置基于POJO的消费者。  
如果我们没有显式地定义 `KafkaListenerContainerFactory` ，Springboot 会自动配置一个默认的 `KafkaListenerContainerFactory` ，其键定义在 `spring.kafka.listener.*`.

如果是用的Spring，需要手动在配置类中添加`@EnableKafka`注解，以便在spring管理的Bean上实现对`@KafkaListener`注解的检测:
```
@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(
          ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, 
          bootstrapAddress);
        props.put(
          ConsumerConfig.GROUP_ID_CONFIG, 
          groupId);
        props.put(
          ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
          StringDeserializer.class);
        props.put(
          ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
          StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> 
      kafkaListenerContainerFactory() {
   
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
          new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

消费消息：
```
@KafkaListener(topics = "topic1", groupId = "foo")
public void listenGroupFoo(String message) {
    System.out.println("Received Message in group foo: " + message);
}
```
我们可以为实现多个消费者监听同一个 Topic，每个监听器都有一个不同的groupId。此外，一个消费者可以监听来自不同 Topic 的消息：
```
@KafkaListener(topics = "topic1, topic2", groupId = "bar")
```

Spring还支持使用监听器中的 `@Header` 注解来检索一个或多个消息头：
```
@KafkaListener(topics = "topicName")
public void listenWithHeaders(
  @Payload String message, 
  @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
      System.out.println(
        "Received Message: " + message"
        + "from partition: " + partition);
}
```

#### 消费特定partition的信息
对于一个有多个分区的主题，`@KafkaListener` 可以指定显示地指定从一个 Topic 的特定分区的初始位移处开始消费：
```
@KafkaListener(
  topicPartitions = @TopicPartition(topic = "topicName",
  partitionOffsets = {
    @PartitionOffset(partition = "0", initialOffset = "0"), 
    @PartitionOffset(partition = "3", initialOffset = "0")}),
  containerFactory = "partitionsKafkaListenerContainerFactory")
public void listenToPartition(
  @Payload String message, 
  @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
      System.out.println(
        "Received Message: " + message"
        + "from partition: " + partition);
}
```
由于这个消费者的 `initialOffset` 被设置为0，所以每次初始化这个消费者时（项目重新启动时），所有以前消费过的来自0和3分区的信息都会被重新消费。

如果我们不需要设置偏移量，我们可以使用`@TopicPartition`注解的`partitions`属性，只设置没有偏移量的分区：
```
@KafkaListener(topicPartitions 
  = @TopicPartition(topic = "topicName", partitions = { "0", "1" }))
```

#### 添加消息过滤器
我们可以通过添加自定义过滤器来配置消费者以消费特定的消息内容。这可以通过给`KafkaListenerContainerFactory`设置一个`RecordFilterStrategy`来完成：
```
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String>
  filterKafkaListenerContainerFactory() {

    ConcurrentKafkaListenerContainerFactory<String, String> factory =
      new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setRecordFilterStrategy(
      record -> record.value().contains("World"));
    return factory;
}
```
然后我们可以配置一个消费者来使用这个容器工厂：
```
@KafkaListener(
  topics = "topicName", 
  containerFactory = "filterKafkaListenerContainerFactory")
public void listenWithFilter(String message) {
    System.out.println("Received Message in filtered listener: " + message);
}
```
所有符合过滤器的信息都将被丢弃。

### Springboot 配置
配置类在 `KafkaProperties` 中：
```
spring:
  kafka:
    bootstrap-servers: "localhost:9092"
    consumer:
      group-id: "myGroup"
      auto-offset-reset: earliest/latest/none
```














