# Producer 设计原理
{docsify-updated}

<center><img src="pics/kafka-producer.png" alt="" width="50%"></center>

大体上来说，用户首先构建待发送的消息对象 `ProducerRecord` ，然后调用 `KafkaProducer#send` 方法进行发送。 `KafkaProducer` 接收到消息后首先对其进行序列化，然后结合本地缓存的元数据信息一起发送给 `partitioner` 去确定目标分区，最后追加写入内存中的消息缓冲地（accumulator）。 此时 `KafkaProducer#send` 方法成功返回。

`KafkaProducer` 中还有一个专门的 `Sender` IO线程负责将缓冲池中的消息分批次发送给对应的broker，完成真正的消息发送逻辑。

## 序列化＋计算目标分区。
<center><img src="pics/producer-1.png" width="70%"></center>

一条所属topic是"test"，消息体是"message"的消息被序列化之后结合 `KafkaProducer` 缓存的元数据（比如该topic分区数信息等）共同传给后面的 `Partitioner` 实现类
进行目标分区的计算。

## 追加写入消息缓冲区（accumulator）。
<center><img src="pics/producer-2.png" width="70%"></center>

`producer` 创建时会创建一个默认32MB（由 `buffer.memory` 参数指定）的 `accumulator` 缓冲区，专门保存待发送的消息。除了之前在“关键参数”部分中提到的 `linger.ms` 和 `batch.size` 等参数之外，该数据结构中还包含了一个特别重要的集合信息：消息批次信息（batches）。该集合本质上是一个 `HashMap` ，里面分别保存了每个topic分区下的batch队列，即前面说的批次是按照topic分区进行分组的。这样发往不同分区的消息保存在对应分区下的batch队列中。举一个简单的例子，假设消息M1、M2被发送到test的0分区但属于不同的batch，M3被发送到test的1分区，那么batches中包含的信息就是 `{"test-0"->[batch1，batch2]，"test-1"->[batch3]}` 。

单个topic分区下的batch队列中保存的是若干个消息批次，每个batch中最重要的3个组件如下。
+ `compressor` ：负责执行追加写入操作。
+ `batch缓冲区` ：由batch.size参数控制，消息被真正追加写入的地方。
+ `thunks` ：保存消息回调逻辑的集合。

这一步执行完毕之后理论上讲 `KafkaProducer.send` 方法就执行完毕了，用户主线程所做的事情就是等待 `Sender` 线程发送消息并执行返回结果回调（CB-CallBack）。

## `Sender` 线程预处理及消息发送。
<center><img src="pics/producer-3.png" width="80%"></center>

此时，该Sender线程登场了。严格来说，Sender线程自KafkaProducer创建后就一直都在运行着。它的工作流程基本上是如下这样的:
1. 不断轮询缓冲区寻找已做好发送准备的分区。
2. 将轮询获得的各个batch按照目标分区所在的leaderbroker进行分组。
3. 将分组后的batch通过底层创建的Socket连接发送给各个broker。
4. 等待服务器端发送response回来。

## `Sender` 线程处理response.
<center><img src="pics/producer-4.png" width="60%"></center>

`Sender` 线程会发送 `PRODUCE` 请求给对应的broker，broker处理完毕之后发送对应的 `PRODUCE` response。一旦 `Sender` 线程接收到response，将依次（按照消息发送顺序）调用batch中的回调方法。

做完这一步，producer发送消息的工作就可以算作100%完成了。通过这4步我们可以看到新版本producer发送事件完全是异步过程。因此在调优producer前我们需要搞清楚性能瓶颈到底是在用户主线程上还是在 `Sender` 线程上。