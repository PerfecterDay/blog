# grpc入门
{docsify-updated}


### gRpc 简介
RPC 只是一种方法调用模型，只要我们能从本地调用远程服务，就可以说这是某种类型的 RPC。  
由于客户端和服务可以通过不同的协议（如 UDP、TCP、HTTP、HTTP/2）进行通信，因此 RPC 也有不同的类型。  
Protobuf 和 JSON、XML 一样，都是数据交换格式，只不过 json 和 xml 基于文本，而 protobuf 基于字节，可以节省大量带宽和序列化时间。  

因此，我们可以将这两者结合起来，形成各种 RPC。例如，通过 UDP 发送 JSON 数据是 RPC 的一种形式，通过 TCP 发送 XML 数据是另一种形式，通过 UDP 发送 Protobuf 数据是另一种形式。  
GRPC 是一种通过 HTTP/2 协议发送 Protobuf 数据的 RPC。

go-micro 是一个微服务框架，服务通过 RPC 进行通信，因此 go-micro 为开发者提供了各种 RPC 供选择，它们以插件的形式出现，如 UDP、HTTP、GRPC 等。  
GRPC 是一种通信模型，可以用 C++、JAVA、Go 等大多数语言实现。因此，Go-GRPC 就是一个用 go 语言编写的 GRPC 插件。

gRPC 可以使用 `protocol buffers` 作为其接口定义语言（IDL）和底层消息交换格式。

在 gRPC 中，客户端应用程序可以直接调用不同机器上服务器应用程序的方法，就像调用本地对象一样，这样就能更轻松地创建分布式应用程序和服务。与许多 RPC 系统一样，gRPC 基于定义服务的理念，指定了可远程调用的方法及其参数和返回类型。在服务器端，服务器实现这一接口并运行 gRPC 服务器来处理客户端调用。在客户端，客户端有一个存根（在某些语言中称为客户端），提供与服务器相同的方法。

gRPC 客户端和服务器可以在各种环境中运行并相互对话--从谷歌内部的服务器到你自己的桌面--而且可以用任何一种 gRPC 支持的语言编写。因此，举例来说，你可以轻松地用 Java 创建一个 gRPC 服务器，用 Go、Python 或 Ruby 编写客户端。此外，最新的 Google 应用程序接口（API）也将拥有 gRPC 版本的接口，让您可以轻松地在应用程序中构建 Google 功能。
<center><img src="pics/grpc.svg"></center>

gRPC 使用带有特殊 gRPC 插件的 protoc 从 proto 文件中生成代码：你会得到生成的 gRPC 客户端和服务器代码，以及用于填充、序列化和检索消息类型的常规协议缓冲区代码。

### gRpc实现的一般步骤
1. 定义调用服务

    搭建gRPC的第一步就是要定义gRPC调用服务的请求和响应类型，通常使用 `protocol buffuers` 的 .proto 文件来定义。
    首先，使用 `service`  关键字定义一个服务，这个服务类似于Java 中的接口，可以在里边定义一个或多个方法。
    ```
    service HelloService {
		rpc SayHello (HelloRequest) returns (HelloResponse);
	}

	message HelloRequest {
		string greeting = 1;
	}

	message HelloResponse {
		string reply = 1;
	}
    ```
    然后，可以在服务内部定义 rpc 方法以及请求和响应的类型。gRPC 支持4种 rpc 方法定义：
      + 简单rpc:客户端向服务器发送一个请求，并得到一个响应，就像普通函数调用一样。
      ```
      	rpc SayHello(HelloRequest) returns (HelloResponse);
      ```
      + 服务器流式 RPC: 是指客户端向服务器发送请求，并获取一个流来读取一系列信息。gRPC 保证了单个 RPC 调用中的消息排序。
      ```
      rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
      ```
	  + 客户端流 RPC：客户端写入一系列信息并发送给服务器，同样使用提供的流。客户端写完信息后，等待服务器读取并返回响应。gRPC 再次保证了单个 RPC 调用中的消息排序。
	  ```
	  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
	  ```
	  + 双向流 RPC:双方使用读写流发送一系列信息。这两个流是独立运行的，因此客户端和服务器可以按照自己喜欢的顺序读写：例如，服务器可以等收到所有客户端消息后再写响应，也可以交替读取消息后再写消息，或者时使用其他读写组合。每个信息流中的信息顺序都会被保留。
	  ```
	  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
	  ```
	最后，要定义好 rpc 方法中的请求和响应类型，上面的 `HelloRequest` 和 `HelloResponse`。
	
2. 生成客户端和服务器端代码  

	从 .proto 文件中的服务定义开始，gRPC 提供`protocol buffuers`编译器插件，生成客户端和服务器端代码。gRPC 用户通常在客户端调用这些应用程序接口，并在服务器端实现相应的应用程序接口。
	+ 在服务器端，服务器会实现RPC服务声明的方法，并运行 gRPC 服务器来处理客户端调用。gRPC 基础架构对传入请求进行解码，执行服务方法，并对服务响应进行编码。
	+ 在客户端，客户端有一个被称为存根（stub）的本地对象（对于某些语言，首选术语是客户端），它实现了与RPC服务相同的方法。然后，客户端只需在本地对象上调用这些方法，这些方法会将调用参数封装在适当的`protocol buffuers`消息类型中，向服务器发送请求，并返回服务器的`protocol buffuers`响应。

#### 服务端实现与启动
1. 服务实现
   ```
	服务端：
    public class TradeService extends GreeterGrpc.GreeterImplBase {
		public void handle(Request request, StreamObserver<Response> responseObserver) {}
	}

	客户端：
    @GrpcClient("tradeService")
    private GreeterGrpc.GreeterBlockingStub tradeService;
   ```

2. 服务器启动  
   使用 `ServerBuilder` 构建并启动服务器。
	1. 使用构建器的 `forPort()` 方法指定用于监听客户端请求的地址和端口。
	2. 创建服务实现类 `TradeService` 的实例，并将其传递给构建器的 `addService()` 方法。
	3. 调用构建器的 `build()` 和 `start()` 为我们的服务创建并启动 RPC 服务器。
	```
	public RouteGuideServer(ServerBuilder<?> serverBuilder, int port, Collection<Feature> features) {
		this.port = port;
		server = serverBuilder.addService(new RouteGuideService(features))
			.build();
	}
	```

#### 客户端实现
要调用服务方法，我们首先需要创建一个stub，或者说两个stub：

+ 阻塞/同步存根：这意味着 RPC 调用等待服务器响应，要么返回响应，要么引发异常。
+ 非阻塞/异步存根：对服务器进行非阻塞调用，异步返回响应。只有使用异步存根，才能进行某些类型的流式调用。

首先，我们需要为stub创建一个 gRPC 通道，指定要连接的服务器地址和端口：
```
public RouteGuideClient(String host, int port) {
  this(ManagedChannelBuilder.forAddress(host, port).usePlaintext());
}

/** Construct client for accessing RouteGuide server using the existing channel. */
public RouteGuideClient(ManagedChannelBuilder<?> channelBuilder) {
  channel = channelBuilder.build();
  blockingStub = RouteGuideGrpc.newBlockingStub(channel);
  asyncStub = RouteGuideGrpc.newStub(channel);
}
```
我们使用 `ManagedChannelBuilder` 来创建通道。

现在，我们可以使用该通道，使用从 `.proto` 文件生成的 `RouteGuideGrpc` 类中提供的 `newStub` 和 `newBlockingStub` 方法创建存根。
```
blockingStub = RouteGuideGrpc.newBlockingStub(channel)；
asyncStub = RouteGuideGrpc.newStub(channel)；
```

有了 stub ，我们就可以调用 gRpc 服务了。

### grpcurl工具
```
grpcurl --plaintext localhost:9090 list
grpcurl --plaintext localhost:9090 list GreeterService
grpcurl --plaintext -d '{"name": "aaa"}' localhost:9090 GreeterService.sayHello

发送ByteString 格式数据时，需要Base64编码：
grpcurl --plaintext -d '{"msgno":1001,"json":{ "req": [ { "code": "1645", "params": { "username": "700210", "custermer_code": "700210", "nulltoken": "", "tradetoken": "", "password": "G2222222", "language": "2" }, "params_ext": {} } ] }}' 10.187.144.42:8101 grpc_client.Greeter.handle
                                |
                                |
                                V
grpcurl --plaintext -d '{"msgno":1001, "json": "eyAicmVxIjogWyB7ICJjb2RlIjogIjE2NDUiLCAicGFyYW1zIjogeyAidXNlcm5hbWUiOiAiNzAwMjEwIiwgImN1c3Rlcm1lcl9jb2RlIjogIjcwMDIxMCIsICJudWxsdG9rZW4iOiAiIiwgInRyYWRldG9rZW4iOiAiIiwgInBhc3N3b3JkIjogIkcyMjIyMjIyIiwgImxhbmd1YWdlIjogIjIiIH0sICJwYXJhbXNfZXh0Ijoge30gfSBdIH0="}' 10.187.144.42:8101 grpc_client.Greeter.handle

grpcurl --plaintext -d '{"msgno":1001, "json": "eyAicmVxIjogWyB7ICJjb2RlIjogIjE2NDUiLCAicGFyYW1zIjogeyAidXNlcm5hbWUiOiAiNzAwMjEwIiwgImN1c3Rlcm1lcl9jb2RlIjogIjcwMDIxMCIsICJudWxsdG9rZW4iOiAiIiwgInRyYWRldG9rZW4iOiAiIiwgInBhc3N3b3JkIjogIkcyMjIyMjIyIiwgImxhbmd1YWdlIjogIjIiIH0sICJwYXJhbXNfZXh0Ijoge30gfSBdIH0="}' 127.0.0.1:8101 grpc_client.Greeter.handle

grpcurl --plaintext -d '{"msgno":2, "json": ""}' 127.0.0.1:8101 grpc_client.Greeter.handle
grpcurl --plaintext -d '{ "msgno":1001, "json": { "req": "[ { "code": "1645", "params": { "username": "700210", "custermer_code": "700210", "nulltoken": "", "tradetoken": "", "password": "G2222222", "language": "2" }, "params_ext": {} } ]" } }' 10.187.144.42:8101 grpc_client.Greeter.handle

grpcurl --plaintext -d '' localhost:8101 grpc_client.Greeter.handle
```

### grpc注册consul
consul health check 和 GRPC 服务注册端口区分开：
```
spring:
  application:
    name: trade-center-service
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        register: true
        port: ${grpc.server.port}
        healthCheckUrl: "http://10.176.81.23:${server.port}"
        health-check-critical-timeout: 3m
        heartbeat:
          enable: true
```

### 常见错误

Protobuf 版本不一致：
```
org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.NoSuchMethodError: com.google.protobuf.GeneratedMessageV3.isStringEmpty(Ljava/lang/Object;)Z
```
解决：
```
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<exclusions>
		<exclusion>
			<groupId>com.google.protobuf</groupId>
			<artifactId>protobuf-java</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```