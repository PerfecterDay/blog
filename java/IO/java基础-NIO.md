# java基础-NIO
{docsify-updated}

## 流与块的比较
面向流 的 I/O 系统一次一个字节地处理数据。一个输入流产生一个字节流的数据，一个输出流消费一个字节流的数据。为流式数据创建过滤器非常容易。链接几个过滤器，以便每个过滤器只负责单个复杂处理机制的一部分，这样也是相对简单的。不利的一面是，面向流的 I/O 通常相当慢。

一个 面向块 的 I/O 系统以块的形式处理数据。每一个操作都在一步中产生或者消费一个数据块。按块处理数据比按(流式的)字节处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

## 通道
实际上 `ChanneI` 可以分为两大类 ： 分别是用于网络读写的 `SelectableChannel` 和用于文件操作的 `FileChannel` 。

`Channel` 是一个对象，可以通过它读取和写入数据。拿 NIO 与原来的 I/O 做个比较，通道就像是流。 所有数据都通过 `Buffer` 对象来处理。永远不会将字节直接写入通道中，相反，是将数据写入包含一个或者多个字节的缓冲区。同样，也不会直接从通道中读取字节，而是将数据从通道读入缓冲区，再从缓冲区获取这个字节。 
通道与流的不同之处在于通道是双向的。而流只是在一个方向上移动(一个流必须是 `InputStream` 或者 `OutputStream` 的子类)， 而通道可以用于读、写或者同时用于读写。 

常见的Channel有： 
+ `FileChannel`
+ `SelectableChannel`
+ `ServerSocketChannel`
+ `SocketChannel`
+ `DatagramChannel`
+ `Pipe.SinkChannel`
+ `Pipe.SourceChannel`
 
所有 `Channel` 都不应该使用构造器来直接创建，而是：
1. 通过相应 `Channel` 的静态方法 `open` 打开
2. 或者通过传统的节点流 `InputStream/OutputStream` 的 `getChannel` 方法来返回对应的 `Channel` 。不同的节点流获取的 `Channel` 不一样， `FileInputStream/FileOutputStream->FileChannel; PipedInputStream->Pipe.SinkChannel; PipedOutputStream->Pipe.SourceChannel` 。

`Channel` 中最常用的方法：
+ `map` : 将 `Channel` 对应的部分或全部数据映射到 `ByteBuffer`
+ `read` : 有很多重载方法，从 `Channel` 中读取数据到给定 `buffer`
+ `write` : 有很多重载方法，将 `buffer` 中的数据写入到 `Channel`

## NIO 网络编程的一般步骤

### 服务器端打开一个 ServerSocketChannel
为了接收连接，我们需要一个 `ServerSocketChannel` 。事实上，我们要监听的每一个端口都需要有一个 `ServerSocketChannel` 。对于每一个端口，我们打开一个 `ServerSocketChannel` ，不能直接使用 `ServerScocket` 的 `getChannel()` 方法来获得 `ServerSocketChannel` 对象，也不能直接绑定到端口，必须使用 `socket()` 方法获得关联的 `ServerScocket` 对象，然后再用该 `ServerScocket` 对象来绑定端口。如下所示：
```
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking( false );
ServerSocket ss = ssc.socket();
InetSocketAddress address = new InetSocketAddress( 9000 );
ss.bind( address );
```
第一行创建一个新的 `ServerSocketChannel` ，最后三行将它绑定到给定的端口。第二行将 `ServerSocketChannel` 设置为*非阻塞*的 。我们必须对每一个要使用的套接字通道调用这个方法，否则非阻塞 I/O 就不能工作。

### 组册打开的 ServerSocketChannel
下一步是将新打开的 `ServerSocketChannel` 注册到 `Selector` 上。为此我们使用 `SelectableChannel.register()` 方法，如下所示：
```
SelectionKey key = ssc.register(selector, SelectionKey.OP_ACCEPT);
```
`register()` 的第一个参数总是这个 `Selector` 。第二个参数是 `OP_ACCEPT` ，这里它指定我们想要监听 accept 事件，也就是在新的连接建立时所发生的事件。这是适用于 `ServerSocketChannel` 的唯一事件类型。 
请注意对 `register()` 的调用的返回值。 `SelectionKey` 代表这个通道在此 `Selector` 上的这个注册。当某个 `Selector` 通知您某个传入事件时，它是通过提供对应于该事件的 `SelectionKey` 来进行的。`SelectionKey` 还可以用于取消通道的注册。

#### Selector/Reactor
Reactor 模式本质上指的是使用”IO多路复用(IO multiplexing) + 非阻塞IO(non-blocking IO)”的模式。所谓“IO多路复用”，指的就是select/poll/epoll这一系列的多路选择器。它支持线程同时在多个文件描述符上阻塞，并在其中某个文件描述符可读写时收到通知。 **IO复用其实复用的不是IO连接，而是复用线程，让一个thread 能够处理多个连接。**

`Selector` 就是您注册对各种 I/O 事件的兴趣的地方，而且当那些事件发生时，就是这个对象告诉您所发生的事件。

我们关心某个 `Channel` 是否发生了读写或者 `Accept` 事件，就把这个 `Channel` 和相应的事件通过 `Channel` 的 `register` 方法注册到 `selector` 对象上，这样 `selector` 会在这些 `channel` 上发生了你感兴趣的事件时通知你。

所以，我们需要做的第一件事就是创建一个 `Selector` ，通过 `Selector` 类的静态方法 `open()` 来获得系统默认的 `Selector` ：
```
Selector selector = Selector.open();
```

然后，我们将对不同的*通道对象*调用 `register()` 方法，以便注册我们对这些对象中发生的 I/O 事件的兴趣。`register()` 的第一个参数总是这个 `Selector` 。 `Selector` 可以同时监控注册到其上的多个 `SelectableChannel` 的IO状况，是非阻塞IO的核心。一个 `Selector` 有三个 `SelectionKey` 的集合：
1. 所有 `SelectionKey` 的集合：代表了注册在该 `Selector` 上的 `Channel` ，可以通过 `keys()` 获取该集合
2. 所有需要IO处理的 `SelectionKey` 集合：代表了所有监测到的、需要进行IO处理的 `Channel` ，调用 `select()` 方法会检测哪些 `Channel` 需要IO处理，然后通过 `selectedKeys()` 方法可以获取该集合
3. 被取消的 `SelectionKey` 的集合：代表了所有被取消了注册关系的 `Channel` ，程序通常不需要直接访问该集合。

同时， `Selector` 还提供了系列和 `select()` 相关的方法：
+ `int select()` : 监控检测所有注册的 `Channel` ，将那些需要进行IO处理的 `Channel` 所对应的 `SelectionKey` 添加到所有选择的 `SelectionKey` 集合中，返回需要IO处理的 `Channel` 的数量。如果没有需要IO处理的 `Channel` ，该方法将阻塞线程
+ `int select(long timeout)`: 可以设置超时时长的 `select`
+ `int selectNow()`: 立即返回的 `select` ，不会阻塞线程
+ `Selector wakeup()`: 使一个未返回的 `select()` 方法立即返回。

##### SelectionKey
`SelectionKey` 代表的是 `SelectableChannel` 和 `Selector` 之间的注册关系。通过它可以或得对应的 `SelectableChannel` 和 `Selector` :
+ `Selector selector()` : 获取 `Selector`
+ `SelectableChannel channel()` : 获取 `SelectableChannel`

### 内部循环
现在已经注册了我们对一些 I/O 事件的兴趣，下面将进入主循环。使用 `Selector` 的几乎每个程序都像下面这样使用内部循环：
```
while(true){
int num = selector.select();
Set selectedKeys = selector.selectedKeys();
Iterator it = selectedKeys.iterator();
while (it.hasNext()) {
     SelectionKey key = (SelectionKey)it.next();
     // ... deal with I/O event ...
}
```
首先，我们调用 `Selector` 的 `select()` 方法。这个方法会阻塞，直到至少有一个已注册的事件发生。当一个或者更多的事件发生时， `select()` 方法将返回所发生的事件的数量。

接下来，我们调用 `Selector` 的 `selectedKeys()` 方法，它返回发生了事件的 `SelectionKey` 对象的一个集合 。

我们通过迭代 `SelectionKeys` 并依次处理每个 `SelectionKey` 来处理事件。对于每一个 `SelectionKey` ，您必须确定发生的是什么 I/O 事件，以及这个事件影响哪些 I/O 对象。

### 监听新连接
程序执行到这里，我们仅注册了 `ServerSocketChannel` ，并且仅注册它们 `ACCEPT` 事件。为确认这一点，我们对 `SelectionKey` 调用 `readyOps()` 方法，并检查发生了什么类型的事件：
```
if ((key.readyOps() & SelectionKey.OP_ACCEPT)
     == SelectionKey.OP_ACCEPT) {
     // Accept the new connection
     // ...
}
```
可以肯定地说， `readOps()` 方法告诉我们该事件是新的连接。

### 接受新的连接
因为我们知道这个服务器套接字上有一个传入连接在等待，所以可以安全地接受它；也就是说，不用担心 `accept()` 操作会阻塞：
```
ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
SocketChannel sc = ssc.accept();
```
下一步是将新连接的 `SocketChannel` 配置为非阻塞的。而且由于接受这个连接的目的是为了读取来自套接字的数据，所以我们还必须将 `SocketChannel` 注册到 `Selector` 上，如下所示：
```
sc.configureBlocking( false );
SelectionKey newKey = sc.register( selector, SelectionKey.OP_READ );
```
注意我们使用 `register()` 的 `OP_READ` 参数，将 `SocketChannel` 注册用于 **READ** 而不是 **ACCEPT** 新连接。

### 删除处理过的 SelectionKey
在处理 `SelectionKey` 之后，我们几乎可以返回主循环了。但是我们必须首先将处理过的 `SelectionKey` 从选定的键集合中删除。如果我们没有删除处理过的键，那么它仍然会在主集合中以一个激活的键出现，这会导致我们尝试再次处理它。我们调用迭代器的 `remove()` 方法来删除处理过的 `SelectionKey` ：
```
it.remove();
```
现在我们可以返回主循环并接受从一个套接字中传入的数据(或者一个传入的 I/O 事件)了。

### 处理传入的 I/O 数据
当来自一个套接字的数据到达时，它会触发一个 I/O 事件。这会导致在主循环中调用 `Selector.select()` 退出阻塞状态并返回一个或者多个 I/O 事件。这一次， `SelectionKey` 将被标记为 `OP_READ` 事件，如下所示：
```
if ((key.readyOps() & SelectionKey.OP_READ)
     == SelectionKey.OP_READ) {
     // Read the data
     SocketChannel sc = (SocketChannel)key.channel();
     // ...
}
```
与以前一样，我们取得发生 I/O 事件的通道并处理它。在本例中，由于这是一个 echo server，我们只希望从套接字中读取数据并马上将它发送回去。

### 回到主循环
每次返回主循环，我们都要调用 `Selector` 的 `select()` 方法，要么阻塞等待，要么取得一组 `SelectionKey` 。

在现实的应用程序中，您需要通过将通道从 `Selector` 中删除来处理关闭的通道。而且您可能要使用多个线程。创建一个线程池来负责 I/O 事件处理中的耗时部分会更有意义。