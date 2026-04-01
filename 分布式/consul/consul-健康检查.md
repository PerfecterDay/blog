# consul 健康检查机制
{docsify-updated}

> https://developer.hashicorp.com/consul/docs/register/health-check/vm

健康检查是验证服务或节点健康状况的配置。健康检查配置嵌套在 `service` 块中。可以在单独的 `check` 块中为服务定义单个健康检查，也可以在一个检查块中定义多个检查。 Consul支持以下健康检查类型：
1. 脚本检查：调用一个外部应用程序来执行健康检查，以适当的退出代码退出，并可能生成输出。脚本检查是最常见的检查类型之一。
2. HTTP 检查：默认发送一个指定的 GET（可以指定其他请求方法） 请求到指定的 URL，并等待指定的时间。 HTTP 检查是最常见的检查类型之一。
3. TCP 检查: 尝试通过 TCP 连接到 IP 或主机名指定的服务器和端口，并等待指定的时间。
4. UDP 检查: 尝试发送 UDP 包到指定的 IP/主机名和端口，并等待指定的时间。
5. Time-to-live (TTL) 检查：被动等待服务主动发起检查。如果在指定时间内没有收到服务的状态更新，就会进入 `critical` 状态。
6. Docker 检查：Docker 检查依赖于与 Docker 容器打包在一起的外部应用程序，这些应用程序通过 Docker `exec` API 端点触发运行。
7. H2ping 检查：向 http2 的端点发送一个 ping 帧。

## HTTP
HTTP 检查向指定 URL 发送 HTTP 请求，并根据 HTTP 响应代码报告服务健康状况。建议使用 HTTP 检查，而不是使用 cURL 或其他外部进程来检查 HTTP 操作的脚本检查。

```
{
  "check": {
    "id": "api",
    "name": "HTTP API on port 5000",
    "http": "https://localhost:5000/health", //指定URL
    "tls_server_name": "",
    "tls_skip_verify": false,
    "method": "POST", //指定请求方法
    "header": { "Content-Type": ["application/json"] },
    "body": "{\"method\":\"health\"}",
    "interval": "10s", //健康检查间隔
    "timeout": "1s", // 超时时间
    "deregister_critical_service_after": "20s" //超过这个时间，consul 会将服务移除
  }
}
```
HTTP 检查默认发送 GET 请求，但也可以在 `method` 字段中指定其他请求方法。你可以在 `header` 中发送其他头信息。头信息块包含一个键和一个字符串数组，如 `{"x-foo"： ["bar", "baz"]}`。默认情况下，HTTP 检查在 10 秒后超时，但可以在 `timeout` 字段中指定自定义超时值。



```
spring:
  cloud:
    consul:
      discovery:
        health-check-path: /actuator/health
        health-check-interval: 10s
        health-check-critical-timeout: 1m
```
1. 服务宕机，Consul 每 10 秒访问 `/actuator/health` 失败；
2. 失败时 Consul 将该实例/服务标记为 `critical` ；
3. 如果服务 在 1 分钟内恢复，Consul 检查再次成功，会将状态从 `critical` 恢复为 `passing` ；
4. 服务重新可用，对外暴露；
5. 如果服务宕机时间超过 1 分钟（即超过 `deregister_critical_service_after` ），Consul 会将服务从注册中心彻底移除；
6. 此时即使服务恢复，Consul 也不会再自动“重新注册”，因为它已经忘记该服务了；
7. 除非你的应用在恢复时执行了重新注册逻辑

服务重启：正常情况下，服务重启时会注销服务注册，然后启动时又会重新注册服务，所以没有影响
consul重启： consul 重启后会加载之前的注册信息，如果服务可用，健康检查通过，那么服务就会依然可用；如果服务宕机，那么健康检查失败后服务会被注销

## TTL 
```
spring:
  cloud:
    consul:
      discovery:
        heartbeat:
          enabled: true
          ttl: 10s
          reregister-service-on-failure: true
```
在 spring-cloud-starter-consul-discovery 中如果启用了 TTL(heartbeat.enabled: true)，那么就会使用 TTL 健康检查， HTTP的检查将会被忽略。
```
{
    "service:trade-center-service-8100": {
        "Node": "gtjaMac-mini.local",
        "CheckID": "service:trade-center-service-8100",
        "Name": "Service 'trade-center-service' check",
        "Status": "passing",
        "Notes": "",
        "Output": "",
        "ServiceID": "trade-center-service-8100",
        "ServiceName": "trade-center-service",
        "ServiceTags": [],
        "Type": "ttl",
        "Interval": "",
        "Timeout": "",
        "ExposedPort": 0,
        "Definition": {},
        "CreateIndex": 0,
        "ModifyIndex": 0
    },
    "service:trade-center-service-8100-management": {
        "Node": "gtjaMac-mini.local",
        "CheckID": "service:trade-center-service-8100-management",
        "Name": "Service 'trade-center-service-management' check",
        "Status": "passing",
        "Notes": "",
        "Output": "",
        "ServiceID": "trade-center-service-8100-management",
        "ServiceName": "trade-center-service-management",
        "ServiceTags": [
            "management"
        ],
        "Type": "ttl",
        "Interval": "",
        "Timeout": "",
        "ExposedPort": 0,
        "Definition": {},
        "CreateIndex": 0,
        "ModifyIndex": 0
    }
}
```

服务重启：正常情况下，服务重启时会注销服务注册，然后启动时又会重新注册服务，所以没有影响
consul重启： consul 重启后会加载之前的注册信息，如果服务可用，会主动上报心跳信息，那么服务就会依然可用；如果服务宕机，那么TTL检查失败后服务会被注销

## TCP
具体配置需要根据 spring-cloud-consul-discovery 的版本而定，有些版本（比如4.2.0）可能不支持配置文件启用：
```
spring:
  application:
    name: my-tcp-service
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        health-check-protocol: tcp       # 指定为 TCP 检查
        health-check-path: /ignored      # 不用管，TCP 不使用
        health-check-interval: 10s       # 健康检查频率
        health-check-timeout: 3s         # 健康检查超时时间
        instance-id: ${spring.application.name}-${spring.cloud.client.ip-address}-${server.port}
        prefer-ip-address: true
        port: 9000                       # 服务运行的端口（TCP 监听）
```