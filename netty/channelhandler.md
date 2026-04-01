#  ChannelHandler
{docsify-updated}

- [ChannelHandler](#channelhandler)
  - [ChannelInboundHandler](#channelinboundhandler)
  - [ChannelOutboundHandler](#channeloutboundhandler)
    - [ChannelPipeline](#channelpipeline)
    - [ChannelHandlerContext 接口](#channelhandlercontext-接口)

在 `ChannelHandler` 被添加到 `ChannelPipeline` 中或者被从 `ChannelPipeline` 中移除时会调用这些 `ChannelHandler` 的生命周期方法:

|      类型       |                         描述                          |
| :-------------: | :---------------------------------------------------: |
|  `handlerAdded`   | 当把 `ChannelHandler` 添加到 `ChannelPipeline` 中时被调用 |
| `handlerRemoved`  |  当从 `ChannelPipeline` 中移除 `ChannelHandler` 时被调用  |
| `exceptionCaught` |  当处理过程中在 `ChannelPipeline` 中有错误产生时被调用  |

Netty 定义了下面两个重要的 ChannelHandler 子接口：

+ `ChannelInboundHandler` ——处理入站数据以及各种状态变化；
+ `ChannelOutboundHandler` ——处理出站数据并且允许拦截所有的操作

## ChannelInboundHandler

`ChannelInboundHandler` 的方法：

|           类型            |                             描述                             |
| :-----------------------: | :----------------------------------------------------------: |
|     `channelRegistered`     | 当 Channel 已经注册到它的 EventLoop 并且能够处理 I/O 时被调用 |
|    `channelUnregistered`    | 当 Channel 从它的 EventLoop 注销并且无法处理任何 I/O 时被调用 |
|       `channelActive`       | 当 Channel 处于活动状态时被调用；Channel 已经连接/绑定并且已经就绪 |
|      `channelInactive`      |   当 Channel 离开活动状态并且不再连接它的远程节点时被调用    |
|    `channelReadComplete`    |             当Channel上的一个读操作完成时被调用              |
|        `channelRead`        |                当从 Channel 读取数据时被调用                 |
| `ChannelWritabilityChanged` | 当 Channel 的可写状态发生改变时被调用。用户可以确保写操作不会完成得太快（以避免发生 OutOfMemoryError）或者可以在 Channel 变为再次可写时恢复写入。可以通过调用 Channel 的 isWritable()方法来检测Channel 的可写性。与可写性相关的阈值可以通过 Channel.config().setWriteHighWaterMark()和 Channel.config().setWriteLowWaterMark()方法来设置 |
|    `userEventTriggered`     | 当 ChannelnboundHandler.fireUserEventTriggered()方法被调用时被调用，因为一个 POJO 被传经了 ChannelPipeline |

## ChannelOutboundHandler

`ChannelOutboundHandler` 的方法：

| 类型                                                         | 描述                                                |
| ------------------------------------------------------------ | --------------------------------------------------- |
| `bind(ChannelHandlerContext,SocketAddress,ChannelPromise)`     | 当请求将 Channel 绑定到本地地址时被调用             |
| `connect(ChannelHandlerContext,SocketAddress,SocketAddress,ChannelPromise)` | 当请求将 Channel 连接到远程节点时被调用             |
| `disconnect(ChannelHandlerContext,ChannelPromise)`             | 当请求将 Channel 从远程节点断开时被调用             |
| `close(ChannelHandlerContext,ChannelPromise)`                  | 当请求关闭 Channel 时被调用                         |
| `deregister(ChannelHandlerContext,ChannelPromise)`             | 当请求将 Channel 从它的 EventLoop 注销时被调用      |
| `read(ChannelHandlerContext)`                                  | 当请求从 Channel 读取更多的数据时被调用             |
| `flush(ChannelHandlerContext)`                                 | 当请求通过 Channel 将入队数据冲刷到远程节点时被调用 |
| `write(ChannelHandlerContext,Object,ChannelPromise)`           | 当请求通过 Channel 将数据写到远程节点时被调用       |

### ChannelPipeline

+ ChannelPipeline 保存了与 Channel 相关联的 ChannelHandler；
+ ChannelPipeline 可以根据需要，通过添加或者删除 ChannelHandler 来动态地修改；
+ ChannelPipeline 有着丰富的 API 用以被调用，以响应入站和出站事件。

ChannelHandlerContext 使得 ChannelHandler 能够和它的 ChannelPipeline 以及其他的 ChannelHandler 交互。 ChannelHandler 可以通知其所属的 ChannelPipeline 中的下一 个 ChannelHandler，甚至可以动态修改它所属的 ChannelPipeline .

| 类型                                | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| AddFirst/addBefore/addAfter/addLast | 将一个ChannelHandler 添加到ChannelPipeline 中                |
| remove                              | 将一个ChannelHandler 从ChannelPipeline 中移除                |
| replace                             | 将 ChannelPipeline 中的一个 ChannelHandler 替换为另一个 ChannelHandler |

ChannelPipeline 的入站操作:

| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| fireChannelRegistered         | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelRegistered(ChannelHandlerContext)方法 |
| fireChannelUnregistered       | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelUnregistered(ChannelHandlerContext)方法 |
| fireChannelActive             | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelActive(ChannelHandlerContext)方法 |
| fireChannelInactive           | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelInactive(ChannelHandlerContext)方法 |
| fireExceptionCaught           | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的exceptionCaught(ChannelHandlerContext, Throwable)方法 |
| fireUserEventTriggered        | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的userEventTriggered(ChannelHandlerContext, Object)方法 |
| fireChannelRead               | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelRead(ChannelHandlerContext, Object msg)方法 |
| fireChannelReadComplete       | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelReadComplete(ChannelHandlerContext)方法 |
| fireChannelWritabilityChanged | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的channelWritabilityChanged(ChannelHandlerContext)方法 |

ChannelPipeline 的出站操作:

| 方法名称      | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| bind          | 将 Channel 绑定到一个本地地址，这将调用 ChannelPipeline 中的下一个ChannelOutboundHandler 的 bind(ChannelHandlerContext, SocketAddress, ChannelPromise)方法 |
| connect       | 将 Channel 连接到一个远程地址，这将调用 ChannelPipeline 中的下一个ChannelOutboundHandler 的 connect(ChannelHandlerContext, Socket￾Address, ChannelPromise)方法 |
| disconnect    | 将Channel 断开连接。这将调用ChannelPipeline 中的下一个ChannelOutboundHandler 的disconnect(ChannelHandlerContext, Channel Promise)方法 |
| close         | 将 Channel 关闭。这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 close(ChannelHandlerContext, ChannelPromise)方法 |
| deregister    | 将 Channel 从它先前所分配的 EventExecutor（即 EventLoop）中注销。这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 deregister(ChannelHandlerContext, ChannelPromise)方法 |
| flush         | 冲刷Channel所有挂起的写入。这将调用ChannelPipeline中的下一个Channel￾OutboundHandler 的flush(ChannelHandlerContext)方法 |
| write         | 将消息写入 Channel。这将调用 ChannelPipeline 中的下一个 Channel￾OutboundHandler的write(ChannelHandlerContext, Object msg, ChannelPromise)方法。注意：这并不会将消息写入底层的 Socket，而只会将它放入队列中。要将它写入 Socket，需要调用 flush()或者 writeAndFlush()方法 |
| writeAndFlush | 这是一个先调用 write()方法再接着调用 flush()方法的便利方法   |
| read          | 请求从 Channel 中读取更多的数据。这将调用 ChannelPipeline 中的下一个ChannelOutboundHandler 的 read(ChannelHandlerContext)方法 |

### ChannelHandlerContext 接口

ChannelHandlerContext 代表了 ChannelHandler 和 ChannelPipeline 之间的关联，每当有 ChannelHandler 添加到 ChannelPipeline 中时，都会创建ChannelHandlerContext。ChannelHandlerContext 的主要功能是管理它所关联的 ChannelHandler 和在同一个 ChannelPipeline 中的其他 ChannelHandler 之间的交互。

ChannelHandlerContext 有很多的方法，其中一些方法也存在于 Channel 和 ChannelPipeline 本身上，但是有一点重要的不同。**如果调用 Channel 或ChannelPipeline 上的这些方法，它们将沿着整个 ChannelPipeline 进行传播。而调用位于 ChannelHandlerContext上的相同方法，则将从当前所关联的 ChannelHandler 开始，并且只会传播给位于该ChannelPipeline 中的下一个能够处理该事件的 ChannelHandler。**

ChannelHandlerContext 的 API：

| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| alloc                         | 返回和这个实例相关联的Channel 所配置的 ByteBufAllocator      |
| bind                          | 绑定到给定的 SocketAddress，并返回 ChannelFuture             |
| channel                       | 返回绑定到这个实例的 Channel                                 |
| close                         | 关闭 Channel，并返回 ChannelFuture                           |
| connect                       | 连接给定的 SocketAddress，并返回 ChannelFuture               |
| deregister                    | 从之前分配的 EventExecutor 注销，并返回 ChannelFuture        |
| disconnect                    | 从远程节点断开，并返回 ChannelFuture                         |
| executor                      | 返回调度事件的 EventExecutor                                 |
| fireChannelActive             | 触发对下一个 ChannelInboundHandler 上的channelActive()方法（已连接）的调用 |
| fireChannelInactive           | 触发对下一个 ChannelInboundHandler 上的channelInactive()方法（已关闭）的调用 |
| fireChannelRead               | 触发对下一个 ChannelInboundHandler 上的channelRead()方法（已接收的消息）的调用 |
| fireChannelReadComplete       | 触发对下一个ChannelInboundHandler上的channelReadComplete()方法的调用 |
| fireChannelRegistered         | 触发对下一个 ChannelInboundHandler 上的fireChannelRegistered()方法的调用 |
| fireChannelUnregistered       | 触发对下一个 ChannelInboundHandler 上的fireChannelUnregistered()方法的调用 |
| fireChannelWritabilityChanged | 触发对下一个 ChannelInboundHandler 上的fireChannelWritabilityChanged()方法的调用 |
| fireExceptionCaught           | 触发对下一个 ChannelInboundHandler 上的fireExceptionCaught(Throwable)方法的调用 |
| fireUserEventTriggered        | 触发对下一个 ChannelInboundHandler 上的fireUserEventTriggered(Object evt)方法的调用handler返回绑定到这个实例的 ChannelHandler |
| isRemo                        | 如果所关联的 ChannelHandler 已经被从 ChannelPipeline中移除则返回 true |
| name                          | 返回这个实例的唯一名称pipeline 返回这个实例所关联的 ChannelPipeline |
| read                          | 将数据从Channel读取到第一个入站缓冲区；如果读取成功则触发一个channelRead事件，并（在最后一个消息被读取完成后）通 知 ChannelInboundHandler 的 channelReadComplet(ChannelHandlerContext)方法 |
| write                         | 通过这个实例写入消息并经过 ChannelPipeline                   |
| writeAndFlush                 | 通过这个实例写入并冲刷消息并经过 ChannelPipeline             |