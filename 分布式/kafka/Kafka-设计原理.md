#  Kafka-设计原理
{docsify-updated}

- [Kafka-设计原理](#kafka-设计原理)
	- [副本与 ISR](#副本与-isr)
	- [文件组织](#文件组织)


### 副本与 ISR 
一个 Kafka 分区本质上是一个备份日志，利用多个相同的备份共同提供冗余来保证系统的高可用性。这些备份在 Kafka 中称为副本。kafka 把副本均匀地分配到 broker 上，并从这些副本中挑选一个作为 leader 对外服务，其余副本称为 follower 副本，follower 被动地向 leader 副本请求数据从而保持与 leader 副本的同步。但是，在同步的过程中，有些 follower 因为各种原因（网络延迟/故障或者系统bug等）会落后 leader 的进度。Kafka 把那些与 leader 副本保持一致的副本集合称为 ISR （in-sync replicas)，包括leader副本本身也在ISR 中。

当producer 提交一条消息到到分区中时，只有 ISR 集合中所有分区副本都接收到并写入了，才被视为 “已提交” 状态。由此可见，若 ISR 包含 N个副本，则可以允许 N-1 个副本崩溃而不丢失消息。另外，当 leader 因为一些原因不能提供服务时，只有 ISR 中follower 副本才有资格去竞争当选新的 leader 。


### 文件组织
Kafka 的数据，实际上是以文件的形式存储在文件系统的。Topic 下有 Partition，Partition 下有 Segment，Segment 是实际的一个个文件，Topic 和 Partition 都是抽象概念。

在目录 /{partitionid}/ 下，存储着实际的 Log 文件（即 Segment），还有对应的索引文件。

每个 Segment 文件大小相等，文件名以这个 Segment 中最小的 Offset 命名，文件扩展名是 .log。Segment 对应的索引的文件名字一样，扩展名是 .index。

有两个 Index 文件：
+ 一个是 Offset Index 用于按 Offset 去查 Message。
+ 一个是 Time Index 用于按照时间去查。

<center><img src="pics/kafka-file.jpeg" width="50%"></center>

为了减少索引文件的大小，降低空间使用，方便直接加载进内存中，这里的索引使用稀疏矩阵，不会每一个 Message 都记录下具体位置，而是每隔一定的字节数，再建立一条索引。 

索引包含两部分：
+ BaseOffset：意思是这条索引对应 Segment 文件中的第几条 Message。这样做方便使用数值压缩算法来节省空间。例如 Kafka 使用的是 Varint。
+ Position：在 Segment 中的绝对位置。
查找 Offset 对应的记录时，会先用二分法，找出对应的 Offset 在哪个 Segment 中，然后使用索引，在定位出 Offset 在 Segment 中的大概位置，再遍历查找 Message。