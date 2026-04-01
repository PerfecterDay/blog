#  Netty的组件和设计
{docsify-updated}

+ `Channel` —Socket的抽象对象;
+ `EventLoop` —控制流、多线程处理、并发;
+ `PipelineLine` -对网络IO的处理器链（容器）
+ `ChannelHandler` -IO处理器，处理Socket收发的数据
+ `ChannelFuture` —Channel完成一些动作后的异步通知

## Channel 接口
基本的 I/O 操作（`bind()、connect()、read()和 write()`）依赖于底层网络传输所提供的原语。在基于 Java 的网络编程中，其底层基本的构造是 `Socket` 。Netty 的 Channel 接口所提供的 API，大大地降低了直接使用 Socket 类的复杂性。 `Channel` 是 Socket 在 netty 中的高层抽象。Netty 提供了一些与定义的类：
+ `EmbeddedChannel`
+ `LocalServerChannel`
+ `NioDatagramChannel`
+ `NioSctpChannel`
+ `NioSocketChannel`

Netty 的 Channel 实现是**线程安全**的，因此你可以存储一个到 Channel 的引用，并且每当你需要向远程节点写数据时，都可以使用它，即使当时许多线程都在使用它。

Channel 的方法：
<center><img src="pics/channel.jpg" width="50%"></center>

`Channel` 一般分为 `ServerChannel` 和一般 `Channel` ， `ServerChannel` 用来监听客户端连接请求并创建一个新的 `Channel` 以处理该连接。 `Channel` 的生命周期状态:

|        状态         |                             描述                             |
| :-----------------: | :----------------------------------------------------------: |
| ChannelUnregistered |          Channel 已经被创建，但还未注册到 EventLoop          |
|  ChannelRegistered  |               Channel 已经被注册到了 EventLoop               |
|    ChannelActive    | Channel 处于活动状态（已经连接到它的远程节点）。它现在可以接收和发送数据了 |
|   ChannelInactive   |                  Channel 没有连接到远程节点                  |

Netty 的 `Channel` 实现是线程安全的，因此你可以存储一个到 `Channel` 的引用，并且每当你需要向远程节点写数据时，都可以使用它，即使当时许多线程都在使用它。下述代码展示了一个多线程写数据的简单例子。需要注意的是，消息将会被保证按顺序发送。
```
final Channel channel =...
final ByteBuf buf = Unpooled.copiedBuffer("your data",CharsetUtil.UTF_8).retain();

Runnable writer = new Runnable() {
	@Override
	public void run() {
		channel.writeAndFlush(buf.duplicate());
	}
};

Executor executor = Executors.newCachedThreadPool();
executor.execute(writer);
executor.execute(writer);
```

## EventLoop
`EventLoop` 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。下图大体上说明了 `Channel` `、EventLoop` `、Thread` 以及 `EventLoopGroup` 之间的关系：

<center><img src="pics/event-loop.jpg" width="40%"></center>

+ 一个 EventLoopGroup 包含一个或者多个 EventLoop；
+ 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定；
+ 所有由 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理；
+  一个 Channel 在它的生命周期内只注册于一个 EventLoop；
+  一个 EventLoop 可能会被分配给一个或多个 Channel。

一个 `EventLoop` 可以处理多个 `Channel` ，所以也是IO多路复用的封装。在这种设计中，一个给定 Channel的 I/O 操作都是由相同的 Thread执行的，实际上消除了对于同步的需要。

## ChannelHandler

`ChannelHandler` 和 `ChannelPipeline` 是管理数据流以及执行应用程序处理逻辑的组件。

### ChannelHandler 接口

从应用程序开发人员的角度来看，Netty 的主要组件是 `ChannelHandler` ，它充当了所有 处理入站和出站数据的应用程序逻辑的容器。这是可行的，因为 `ChannelHandler` 的方法是由网络事件(其中术语“事件”的使用非常广泛)触发的。
<center><img src="pics/channel-handler.jpg" width="50%" ></center>

Netty 以适配器类的形式提供了大量默认的 `ChannelHandler` 实现，其旨在简化应用程序处理逻辑的开发过程。你已经看到了， `ChannelPipeline` 中的每个 `ChannelHandler` 将负责把事件转发到链中的下一个 `ChannelHandler` 。这些适配器类(及它们的子类)将自动执行这个操作，所以你可以只重写那些你想要特殊处理的方法和事件。  

下面这些是编写自定义 ChannelHandler 时经常会用到的适配器类:
+ `ChannelHandlerAdapter`
+ `ChannelInboundHandlerAdapter` 
+ `ChannelOutboundHandlerAdapter`
+ `ChannelDuplexHandler`

### 编码器和解码器

当通过 Netty 发送或者接收一个消息的时候，就将会发生一次数据转换。入站消息会被**解码**:也就是说，从字节转换为另一种格式，通常是一个 Java 对象。如果是出站消息，则会发生相反方向的转换：它将从它的当前格式被**编码**为字节。这两种方向的转换的原因很简单：网络数据总是一系列的字节。

所有由 Netty 提供的编码器/解码器适配器类都实现了 `ChannelOutboundHandler` 或者 `ChannelInboundHandler` 接口。

## ChannelPipeline 接口

`ChannelPipeline` 提供了 `ChannelHandler` 链的容器，并定义了用于在该链上传播入站和出站事件流的 API。当 `Channel` 被创建时，它会被自动地分配到它专属的 `ChannelPipeline`， `ChannelHandler` 安装到 `ChannelPipeline` 中的过程如下所示：
1. 一个 `ChannelInitializer` 的实现被注册到了 `ServerBootstrap` 中
2. 当 `ChannelInitializer.initChannel()` 方法被调用时， `ChannelInitializer` 将在 `ChannelPipeline` 中安装一组自定义的 `ChannelHandler`
3. `ChannelInitializer` 将它自己从 `ChannelPipeline` 中移除。

`ChannelPipeline` 可以根据需要，通过添加或者删除 `ChannelHandler` 来动态地修改处理逻辑。
<center><img src="pics/channel-pipeline.jpg" width="50%" ></center>

如果一个消息或者任何其他的入站事件被读取，那么它会从 `ChannelPipeline` 的头部开始流动，并被传递给第一个 `ChannelInboundHandler` 。这个 `ChannelHandler` 不一定会实际地修改数据，具体取决于它的具体功能，在这之后，数据将会被传递给链中的下一个 `ChannelInboundHandler` 。最终，数据将会到达 `ChannelPipeline` 的尾端，届时，所有处理就都结束了。  
数据的出站运动(即正在被写的数据)在概念上也是一样的。在这种情况下，数据将从 `ChannelOutboundHandler` 链的尾端开始流动，直到它到达链的头部为止。在这之后，出站数据将会到达网络传输层，这里显示为 Socket。通常情况下，这将触发 `socket` 一个写操作。

在Netty中，有两种发送消息的方式。你可以直接写到 `Channel` 中，也可以 写到和 `ChannelHandler` 相关联的 `ChannelHandlerContext` 对象中。前一种方式将会导致消息从 `ChannelPipeline` 的尾端开始流动，而后者将导致消息从 `ChannelPipeline` 中的下一个 `ChannelHandler` 开始流动。

如果 `ChannelHandler` 是一个耗时很长的任务，因为一个 `EventLoop` 可能会绑定很多 `Channel`，那么这个长时间处理的任务可能会阻塞后续的 `Channel` 事件的处理。如下图所示：
<center><img src="pics/netty_thread_model2.png" width="80%"></center>

用户在向 `ChannelPipeline` 添加处理程序时，可以指定一个 `EventExecutor/EventExecutorGroup` 。 如果指定， `ChannelHandler` 的处理程序方法将始终由指定的 `EventExecutor/EventExecutorGroup` 中的线程调用。 如果未指定，处理程序方法将始终由其关联 `Channel` 注册的 `EventLoop` 调用。

为了不阻塞其它 `Channel` 事件的处理，我们需要指定 `EventExecutorGroup/EventExecutor`。 这种情况下，同一个 `ChannelHandler` 始终由同一线程调用，如果指定了多线程的 `EventExecutorGroup` ，则会首先选择其中一个线程将 `ChannelHandler` 与其绑定，然后一直使用所选线程执行直至注销。 如果同一 `Channel` 中的两个 `ChannelHandler` 分配给不同的 `EventExecutorGroup`，那么他们会并发调用。 需要注意并发安全问题。

## 线程模型
> https://stackoverflow.com/questions/65345371/netty-thread-model-vs-long-running-taks

<center>
<img src="pics/netty_thread_model.png" alt="">

</center>
