# envoy 日志
{docsify-updated}

## Access Log
> https://www.envoyproxy.io/docs/envoy/v1.38.3/configuration/observability/access_log/usage.html

### `%RESPONSE_FLAGS%`
如果有的话，表示响应或者连接的附加详情。 对于 TCP 连接，说明中提到的响应码不适用。 可能的值如下：

**客户端 / 下游相关错误 (Downstream)**
+ DC（DownstreamConnectionTermination）: 下游连接中断。
+ DPE (Downstream Protocol Error): 下游请求存在一个 HTTP 协议错误。

**上游相关错误 (Upstream / 后端服务)**
+ UF（UpstreamConnectionFailure）: 附加在 503 响应状态码，表示上游连接失败
+ UT（UpstreamRequestTimeout）: 附加在 504 响应状态码，上游请求超时。
+ URX（UpstreamRetryLimitExceeded）: 请求因为达到了上游重试限制 (HTTP) 或者最大连接尝试 (TCP) 而被拒绝。
+ UPE (Upstream Protocol Error)：上游后端服务返回了不符合 HTTP 协议规范的数据。
+ UC（UpstreamConnectionTermination）: 附加在 503 响应状态码，上游连接中断。
+ DT(DurationTimeout): 连接超时

**路由与配置错误 (Routing / Configuration)**
+ NR (NoRouteFound): 附加在 404 响应状态码，表示无给定请求的 路由配置 ，或者对于一个下游连接没有匹配的过滤器链。
+ UH (NoHealthyUpstream): 附加在 503 响应状态码，表示上游集群中无健康的上游主机。
+ NC (NoClusterFound)：请求匹配到了路由，但是路由指向的 Cluster（上游集群）不存在。

**速率限制与熔断 (Rate Limiting / Circuit Breaking)**
+ RL (Rate Limited)： 429 相应状态码，请求触发了 Envoy 的本地速率限制（Local Rate Limit）或全局速率限制（Global Rate Limit Filter）而被拒绝。
+ UO (UpstreamOverflow): 附加在 503 响应状态码，表示上游溢出 (熔断) 。

**本地错误与拒绝 (Local Errors / Security)**
+ LH（FailedLocalHealthCheck）: 附加在 503 响应状态码，本地服务 健康检查请求 失败。
+ LR（LocalReset）：Connection local reset in addition to 503 response code.
+ UR (UpstreamRemoteReset): Upstream remote reset in addition to 503 response code.
+ OM (Overload Manager)：Envoy 的内存或资源占用过高，触发了高级别过载管理器（Overload Manager）的主动丢包/拒绝策略。
+ SI (Stream Idle Timeout)：连接因为长时间没有数据传输，触发了流空闲超时（Stream Idle Timeout）而被 Envoy 主动关闭。
+ FI (Fault Injected)：请求被 Envoy 配置的故障注入（Fault Injection）拦截（例如人为模拟的延迟或 503 注入，用于混沌工程测试）。
+ RLSE (Rate Limit Service Error)：速率限制服务（Rate Limit Service）本身发生故障，且配置为失败时拒绝请求。

### %START_TIME%
+ `HTTP/THRIFT`: Request start time including milliseconds.
+ `TCP`: Downstream connection start time including milliseconds.
+ `UDP`: UDP proxy session start time including milliseconds.

```
%START_TIME(%Y/%m/%dT%H:%M:%S%z)%
%START_TIME(%s)%

# To include millisecond fraction of the second (.000 ... .999). E.g. 1527590590.528.
%START_TIME(%s.%3f)%
%START_TIME(%s.%6f)%
%START_TIME(%s.%9f)%
```

### %COMMON_DURATION(START:END:PRECISION)%
在特定 PRECISION 下的 START 时间点与 END 时间点之间的总持续时间。START 和 END 时间点由以下值指定（此处所有值均区分大小写）：
+ `DS_RX_BEG`: The time point of the downstream request receiving begin.
+ `DS_RX_END`: The time point of the downstream request receiving end.
+ `US_CX_BEG`: The time point of the upstream TCP connect begin.
+ `US_CX_END`: The time point of the upstream TCP connect end.
+ `US_HS_END`: The time point of the upstream TLS handshake end.
+ `US_TX_BEG`: The time point of the upstream request sending begin.
+ `US_TX_END`: The time point of the upstream request sending end.
+ `US_RX_BEG`: The time point of the upstream response receiving begin.
+ `US_RX_BODY_BEG`: The time point of the upstream response body receiving begin.
+ `US_RX_END`: The time point of the upstream response receiving end.
+ `DS_TX_BEG`: The time point of the downstream response sending begin.
+ `DS_TX_END`: The time point of the downstream response sending end.
+ Dynamic value: Other values will be treated as custom time points that are set by named keys.

整个时间序列如下
```
Client                         Envoy                         Upstream
  │                              │                              │
  │                              │                              │
  │──── HTTP Request ───────────>│                              │
  │                              │                              │
  │                         DS_RX_BEG                            │
  │                              ●                              │
  │                              │                              │
  │                    接收 Client 请求                         │
  │                              │                              │
  │                         DS_RX_END                            │
  │                              ●                              │
  │                              │                              │
  │                              │
  │                              │──── TCP connect ────────────>│
  │                              │                              │
  │                              │ US_CX_BEG                    │
  │                              ●                              │
  │                              │                              │
  │                              │ US_CX_END                    │
  │                              ●<──────── TCP connected ──────│
  │                              │                              │
  │                              │──── TLS handshake ──────────>│
  │                              │                              │
  │                              │ US_HS_END                    │
  │                              ●<──────── TLS done ──────────│
  │                              │                              │
  │                              │
  │                              │ US_TX_BEG                    │
  │                              ●                              │
  │                              │──── Request ────────────────>│
  │                              │                              │
  │                              │ US_TX_END                    │
  │                              ●                              │
  │                              │                              │
  │                              │<──── Response Headers ───────│
  │                              │                              │
  │                              │ US_RX_BEG                    │
  │                              ●                              │
  │                              │                              │
  │                              │<──── Response Body ──────────│
  │                              │                              │
  │                              │ US_RX_END                    │
  │                              ●                              │
  │                              │                              │
  │                              │
  │<──────── Response ───────────│                              │
  │                              │                              │
  │                         DS_TX_BEG                            │
  │                              ●                              │
  │                              │                              │
  │                         DS_TX_END                            │
  │                              ●                              │
  │                              │                              │
```

PRECISION 由以下值指定（此处所有值均区分大小写）：
+ `ms`: Millisecond precision.
+ `us`: Microsecond precision.
+ `ns`: Nanosecond precision.

```
%COMMON_DURATION(DS_RX_BEG:DS_RX_END:ms)%  //下游请求接收耗时
%COMMON_DURATION(US_CX_BEG:US_CX_END:ms)%  //上游TCP连接建立耗时
%COMMON_DURATION(DS_RX_BEG:DS_RX_END:ms)%  //下游请求接收耗时
%COMMON_DURATION(DS_RX_BEG:DS_RX_END:ms)%  //下游请求接收耗时
```

相关的一些耗时统计：
+ `%REQUEST_DURATION%`: 从开始时间到从下游接收到的请求的最后一个字节(`DS_RX_END`)，该请求的总持续时间（以毫秒为单位）。
+ `%REQUEST_TX_DURATION%`: 从开始时间到向上游发送的最后一个字节(`US_TX_END`)，该请求的总持续时间（以毫秒为单位）。
+ `%RESPONSE_DURATION%`: 从开始时间到从上游主机读取第一个字节为止(`US_RX_BEG`)，该请求的总持续时间（以毫秒为单位）。
+ `%ROUNDTRIP_DURATION%`: 从开始时间到收到下游发来的最终确认，该请求的总持续时间（以毫秒为单位）。
+ `%RESPONSE_TX_DURATION%`: 从上游主机读取第一个字节到向下游发送最后一个字节，整个请求的总耗时（以毫秒为单位）。
+ `%DOWNSTREAM_HANDSHAKE_DURATION%`: 从开始时间到下游完成 TLS 握手，该请求的总持续时间（以毫秒为单位）。
+ `%UPSTREAM_CONNECTION_POOL_READY_DURATION%`: 从创建上游请求到连接池就绪为止的总耗时（以毫秒为单位）。

### %REQUEST_HEADER(X?Y):Z% / %REQ(X?Y):Z%
打印 HTTP 请求头，其中 X 是主 HTTP 请求头，Y 是备用请求头，Z 是可选参数，表示将字符串截断至 Z 个字符。系统会首先从名为 X 的 HTTP 请求头中获取值；如果该请求头未设置，则使用请求头 Y。如果这两个请求头均不存在，日志中将显示“-”符号。
类似的还有响应头 `%RESPONSE_HEADER(X?Y):Z% / %RESP(X?Y):Z%`


### 其他
+ `%PROTOCOL%`
+ `%UPSTREAM_PROTOCOL%`
+ `%RESPONSE_CODE%`
+ `%DURATION%`: 请求总耗时（毫秒）
+ `%UPSTREAM_REQUEST_ATTEMPT_COUNT%`
+ `%ROUTE_NAME%` 
+ `%UPSTREAM_HOST%`: 上游主机的主地址（例如，TCP 连接中的 IP 地址：端口号）。
+ `%UPSTREAM_HOST_NAME%`: 上游主机的主机名（例如，DNS 名称）。如果没有 DNS 名称，则将使用上游主机的主地址（例如，TCP 连接中的 ip:port）。
+ `%UPSTREAM_CLUSTER%`: 请求的上游集群的名称。
+ `%UPSTREAM_TRANSPORT_FAILURE_REASON%`: 如果上游连接因传输套接字（例如 TLS 握手）而失败，则提供来自传输套接字的失败原因。该字段的格式取决于配置的上游传输套接字。常见的 TLS 故障请参见 TLS 故障排除。
+ `%DOWNSTREAM_TRANSPORT_FAILURE_REASON%`: 如果因传输套接字（例如 TLS 握手）导致下游连接失败，则提供来自传输套接字的失败原因。该字段的格式取决于配置的下游传输套接字。常见的 TLS 故障请参见 TLS 故障排除。
+ `%REQUESTED_SERVER_NAME(X:Y)%`: 在 SSL 连接套接字上设置的字符串值，用于服务器名称指示（SNI）或主机头。参数 X 用于指定当未设置 SNI 时，输出是否应回退到主机头。参数 Y 用于指定请求主机的来源。X 和 Y 均为可选参数。当 X 设置为 SNI_ONLY 时，Y 没有意义。


完整的值： https://www.envoyproxy.io/docs/envoy/v1.38.3/configuration/advanced/substitution_formatter#config-advanced-substitution-operators

### 配置输出日志
```
envoy -c envoy-demo.yaml --log-path logs/custom.log //设置日志输出文件
envoy -c envoy-demo.yaml -l debug --log-path logs/custom.log //设置日志级别
```