# go 与 GRPC 集成
{docsify-updated}

### 环境配置

1. 安装 Protocol Buffer Compiler： `brew install protobuf`
2. 安装 Go plugins for the protocol compiler:
   ```
    $ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
    $ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
   ```
3. 更新 PATH，以便 protoc 编译器能找到插件: `export PATH="$PATH:$(go env GOPATH)/bin"`


### 编译
```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    helloworld/helloworld.proto
```

### 代码示例
```
package main

import (
	"flag"
	"fmt"
	pb "demogitlab.demo.net/demo-app-platform/go-trade-gmt/grpc_client"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
	"log"
	"net"
)

var (
	port = flag.Int("port", 50051, "The server port")
)

// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

func main() {
	//cmd.Execute()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	reflection.Register(s)
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

### 问题
```
大写定义：
rpc Handle (Request) returns (Response) {}

jsonservice.pb.micro.go
func (c *greeterService) Handle(ctx context.Context, in *Request, opts ...client.CallOption) (*Response, error) {
	req := c.c.NewRequest(c.name, "Greeter.Handle", in)
	out := new(Response)
	err := c.c.Call(ctx, req, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

小写定义：
rpc handle (Request) returns (Response) {}
func (c *greeterService) Handle(ctx context.Context, in *Request, opts ...client.CallOption) (*Response, error) {
	req := c.c.NewRequest(c.name, "Greeter.handle", in)
	out := new(Response)
	err := c.c.Call(ctx, req, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

### 原理

1. 服务端会使用生成的 pb.go 文件中的方法和描述去注册服务的信息（服务名/方法名/handler等）
	jsonservice_grpc.pb.go
	```
	func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
		s.RegisterService(&Greeter_ServiceDesc, srv)
	}

	var Greeter_ServiceDesc = grpc.ServiceDesc{
		ServiceName: "grpc_client.Greeter",
		HandlerType: (*GreeterServer)(nil),
		Methods: []grpc.MethodDesc{
			{
				MethodName: "handle",
				Handler:    _Greeter_Handle_Handler,
			},
		},
		Streams:  []grpc.StreamDesc{},
		Metadata: "jsonservice.proto",
	}
	```
	信息都会保存到grpc 的 Sever 结构体中（services）。
	```
	type Server struct {
		opts serverOptions

		mu  sync.Mutex // guards following
		lis map[net.Listener]bool
		// conns contains all active server transports. It is a map keyed on a
		// listener address with the value being the set of active transports
		// belonging to that listener.
		conns    map[string]map[transport.ServerTransport]bool
		serve    bool
		drain    bool
		cv       *sync.Cond              // signaled when connections close for GracefulStop
		services map[string]*serviceInfo // service name -> service info
		events   trace.EventLog

		quit               *grpcsync.Event
		done               *grpcsync.Event
		channelzRemoveOnce sync.Once
		serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop

		channelzID int64 // channelz unique identification number
		czData     *channelzData

		serverWorkerChannels []chan *serverWorkerData
	}
	```

2. grpc服务端收到请求的时候就会根据传入的服务名和方法名去寻找注册的服务和方法并调用
