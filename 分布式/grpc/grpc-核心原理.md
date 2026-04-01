# Grpc 核心概念
{docsify-updated}

> https://grpc.github.io/grpc-java/javadoc/index.html

gRPC 基于远程过程调用的客户-服务器模型。 客户端创建一个与服务器相连的通道。 RPC 由客户端发起并发送给服务器，服务器再回复客户端。 当客户端和服务器发送完信息后，它们会半关闭各自的连接。 服务器一关闭，RPC 就完成了。

客户端：  
要发送 RPC，首先要使用 `ManagedChannelBuilder.forTarget(java.lang.String)` 创建一个通道。 使用自动生成的 Protobuf 存根时，存根类将有用于封装通道的构造函数。 这些构造函数包括 `newBlockingStub`、 `newStub`  和 `newFutureStub` ，你可以根据自己的设计来使用它们，存根是客户端与服务器交互的主要方式。还可以指定负载均衡策略：
```
ManagedChannel channel = Grpc.newChannelBuilder(target, InsecureChannelCredentials.create())
            .defaultLoadBalancingPolicy("round_robin")
        .build();
GreeterGrpc.GreeterBlockingStub blockingStub = GreeterGrpc.newBlockingStub(channel);
```

服务器端：  
要接收 RPC，请使用 `ServerBuilder.forPort(int)` 创建一个服务器。 Protobuf 存根将包含一个名为 `AbstractFoo` 的抽象类，其中 Foo 是你的服务名称。 扩展这个类，并将它的实例传递给 `ServerBuilder.addService(io.grpc.ServerServiceDefinition)` 。 服务器构建完成后，调用 `Server.start()` 开始接受 RPC。

客户端和服务器都应使用自定义执行器。 gRPC 运行时包含一个默认执行器，可以方便测试和示例，但并不适合在生产环境中使用。 

客户端和服务器也可以使用 `shutdown` 方法优雅地关闭。 gRPC 还支持更多高级功能，如名称解析、负载平衡、双向流、健康检查等。 请参阅客户端和服务器构建器中的相关方法。 gRPC 的开发主要在 Github 上进行，网址是 `https://github.com/grpc/grpc-java` ，gRPC 团队欢迎贡献和错误报告。 如果你有关于 gRPC 的问题，也可以在 grpc-io 上查看邮件列表。

```
服务器端：
public class GreeterServiceImpl extends GreeterServiceGrpc.GreeterServiceImplBase {
    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        String name = request.getName();
        HelloReply reply = HelloReply.newBuilder()
            .setMessage("Hello, " + name)
            .build();

        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
}

public static void main(String[] args) throws Exception {
	Server server = ServerBuilder
		.forPort(50051)
		.addService(new GreeterServiceImpl())
		.build();

	server.start();
	System.out.println("Server started on port 50051");

	server.awaitTermination();
}

客户端：
public class ClientApp {
    public static void main(String[] args) {
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("localhost", 50051)
            .usePlaintext()
            .build();

        GreeterServiceGrpc.GreeterServiceBlockingStub stub = GreeterServiceGrpc.newBlockingStub(channel);

        HelloRequest request = HelloRequest.newBuilder().setName("gRPC Java").build();
        HelloReply reply = stub.sayHello(request);

        System.out.println("Received: " + reply.getMessage());

        channel.shutdown();
    }
}
```

## 主要核心类
| 组件名                | 说明                                       | 生命周期阶段       |
|---------------------|------------------------------------------|------------------|
| Server              | gRPC 服务器实例，管理服务注册与监听                    | 启动时初始化       |
| HandlerRegistry     | gRPC 服务器实例，管理服务注册与监听                    | 启动时初始化       |
| ServerServiceDefinition | 描述服务和其方法                               | 注册服务时         |
| ServerCall          | 表示一次 RPC 调用通道（负责 send/close）           | 每次请求创建       |
| ServerCallHandler   | 调用服务实现的桥梁（创建监听器）                   | 每次请求           |
| ServerInterceptor   | 拦截请求，用于认证/日志/上下文绑定                 | 每次请求           |
| Context             | 请求上下文，用于跨组件传值                          | 每次请求           |
| ServerCall.Listener | 监听请求事件（消息到达、关闭）                     | 每次请求           |
| StreamObserver      | 发送响应给客户端                                 | 服务实现中使用     |


## 请求处理的一般流程
1. 请求接入阶段
	+ Server 启动后监听端口，接收到请求后通过注册的服务（ServerServiceDefinition）路由到对应方法。

2. 拦截器链执行
	+ 多个 ServerInterceptor 被依次调用。
	+ 每个拦截器可以：
	+ 检查 metadata
	+ 设置 Context
	+ 包装 ServerCall 或 ServerCall.Listener
	+ 提前拦截/终止请求（调用 call.close()）

3. 创建 Context 并绑定到调用链
	+ 通过 Contexts.interceptCall(context, call, headers, next)，让 gRPC 框架把该 Context 自动激活到 listener 生命周期中。

4. 调用服务方法（通过 Handler 和 Listener）
	+ ServerCallHandler.startCall() 被调用，内部会创建一个 ServerCall.Listener
	+ 后续的 onMessage(), onHalfClose() 等被调用，触发实际业务处理

5. 响应客户端
	+ 服务实现中使用 StreamObserver.onNext() 写响应
	+ 最终通过 onCompleted() 结束
	+ 底层由 ServerCall.sendHeaders()、sendMessage()、close() 完成底层传输

## 核心原理
```
ServerBuilder.build()
  → ServerImpl (构建服务)
      └── start() → NettyServer.start()
                      └── NettyServerHandler (HTTP/2 handler)
                          └── ServerImpl.streamCreated()
                              └── 调用拦截器 + 业务处理
```

启动时：
1. `NettyServer` 的 `start` 方法会初始化 Neety 的处理 pipeline ：`b.childHandler`， 构造 `NettyServerTransport` 对象并调用其 `start()` 方法。
2. `NettyServerTransport` 的 `start` 方法会创建 `NettyServerHandler`，并对其进行包装后添加到当前连接 `channel` 的 pipeline 中以处理请求。
```
public void start(ServerTransportListener listener) {
        Preconditions.checkState(this.listener == null, "Handler already registered");
        this.listener = listener;
        this.grpcHandler = this.createHandler(listener, this.channelUnused); //创建 NettyServerHandler
        ChannelHandler negotiationHandler = this.protocolNegotiator.newHandler(this.grpcHandler);
        ChannelHandler bufferingHandler = new WriteBufferingAndExceptionHandler(negotiationHandler);
		....
        this.channel.pipeline().addLast(new ChannelHandler[]{bufferingHandler});
    }
```

处理请求时：
1. 经过一系列处理，最终调用 `NettyServerHandler` 的方法，`NettyServerHandler` 会调用 `ServerImpl` 的方法。


<center><img src="pics/grpc_call.png" width="60%"></center>

`HandlerRegistry`

