# Springboot 集成 grpc

## grpc-spring-boot-starter 依赖
上述配置过程非常复杂，可以使用开源工具类帮助我们简化这些步骤。

```
gRPC Server + Client
<dependency>
	<groupId>net.devh</groupId>
	<artifactId>grpc-spring-boot-starter</artifactId>
</dependency>

gRPC Server
<dependency>
  <groupId>net.devh</groupId>
  <artifactId>grpc-server-spring-boot-starter</artifactId>
</dependency>

gRPC Client
<dependency>
  <groupId>net.devh</groupId>
  <artifactId>grpc-client-spring-boot-starter</artifactId>
</dependency>
```

## 服务端
1. 服务端配置
	参照 `grpc-server-spring-boot-autoconfigure` jar 包中的 `net.devh.boot.grpc.server.config.GrpcServerProperties` 类
2. `@GrpcClient("myService")` 注解为每一个 RPC 服务调用者注册一个名字，可以在配置文件中为各个调用者定义不同的配置。 `GLOBAL` 代表全局配置。
3. `@GrpcService` 就可以注册一个 gRpc 服务
4. `AnnotationGrpcServiceDiscoverer` 这个类处理了 `@GrpcService` 注解的自动注入


`GrpcServerLifecycle` --> `createAndStartGrpcServer` 等价于我们使用原生 grpc 开发时使用的 ：
```
Server server =
    NettyServerBuilder.forPort(12345)
        .addService(ProtoReflectionService.newInstance()).build();
```


### 配置 ServerInterceptor
向您的服务端添加 ` ServerInterceptor` 的三种方式。

+ 使用 `@GrpcGlobalServerIntercetor` 注解或者使用 `GlobalServerIntercetorConfigurer` 将 `ServerInterceptor` 定义为全局拦截器
+ 在 `@GrpcService#interceptors` 或 `@GrpcService#interceptorNames` 字段中明确列出
+ 使用 ` GrpcServerConfigurer 并调用 serverBuilder.intercept(ServerInterceptor interceptor)` 方法

### GrpcServerConfigurer
Grpc 服务端配置器允许您将自定义配置添加到 gRPC 的 `ServerBuilder` 。
```
@Bean
public GrpcServerConfigurer keepAliveServerConfigurer() {
    return serverBuilder -> {
        if (serverBuilder instanceof NettyServerBuilder) {
            ((NettyServerBuilder) serverBuilder)
                    .keepAliveTime(30, TimeUnit.SECONDS)
                    .keepAliveTimeout(5, TimeUnit.SECONDS)
                    .permitKeepAliveWithoutCalls(true);
        }
    };
}
```
## 客户端
1. 客户端配置参照 `grpc-client-spring-boot-autoconfigure` jar 包中的 `net.devh.boot.grpc.client.config.GrpcChannelsProperties` 类
2. `GrpcClientBeanPostProcessor` 这个类处理使用了 `@GrpcClient` 注解的自动注入


### 服务发现-DiscoveryClientResolverFactory
`DiscoveryClientResolverFactory` 会判断URL的协议类型是不是 `discovery://` ，如果是的话，会使用 `DiscoveryClientNameResolver` 来解析这个 URL。

consul 中的 `ConsulCatalogWatchAutoConfiguration` 会注册一个 `ConsulCatalogWatch` ， `ConsulCatalogWatch` 会启动一个定时任务，这个定时任务会定期去调用 `GET /v1/catalog/services` API 更新服务列表，并且发布 `HeartbeatEvent` 事件通知相关组件。
```
@Timed("consul.watch-catalog-services")
public void catalogServicesWatch() {
    try {
        long index = -1;
        if (this.catalogServicesIndex.get() != null) {
            index = this.catalogServicesIndex.get().longValue();
        }

        CatalogServicesRequest request = CatalogServicesRequest.newBuilder()
                .setQueryParams(new QueryParams(this.properties.getCatalogServicesWatchTimeout(), index))
                .setToken(this.properties.getAclToken()).build();
        Response<Map<String, List<String>>> response = this.consul.getCatalogServices(request);
        Long consulIndex = response.getConsulIndex();
        if (consulIndex != null) {
            this.catalogServicesIndex.set(BigInteger.valueOf(consulIndex));
        }

        if (log.isTraceEnabled()) {
            log.trace("Received services update from consul: " + response.getValue() + ", index: " + consulIndex);
        }
        this.publisher.publishEvent(new HeartbeatEvent(this, consulIndex));
    }
    catch (Exception e) {
        log.error("Error watching Consul CatalogServices", e);
    }
}
```

通过下述配置可以配置 `ConsulCatalogWatch` 的行为：
```
spring:
  cloud:
    consul:
      discovery:
        catalog-services-watch-delay: 1000  # 每隔多少ms去调用consul API
        catalog-services-watch-timeout: 10
```

`grpc-spring-boot-starter` 组件中的 `DiscoveryClientResolverFactory` 会监听 `HeartbeatEvent` 并且更新服务列表：
```
@EventListener(HeartbeatEvent.class)
public void heartbeat(final HeartbeatEvent event) {
    if (this.monitor.update(event.getValue())) {
        for (final DiscoveryClientNameResolver discoveryClientNameResolver : this.discoveryClientNameResolvers) {
            discoveryClientNameResolver.refreshFromExternal();
        }
    }
}
```

### 负载均衡
`GrpcChannelProperties` 中指定的默认负载均衡策略是 `round_robin`:
```
private static final String DEFAULT_DEFAULT_LOAD_BALANCING_POLICY = "round_robin";


grpc:
  client:
    GLOBAL:
      enableKeepAlive: true
      keepAliveWithoutCalls: true
      negotiationType: plaintext
      default-default-load-balancing-policy: round_robin
    tradeService:
      address: 'discovery:///trade-center-service'
  server:
    port: 8912
```
