# envoy 日志
{docsify-updated}

## Access Log
> https://www.envoyproxy.io/docs/envoy/v1.38.3/configuration/observability/access_log/usage.html

1. `%RESPONSE_FLAGS%`
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


完整的值： https://www.envoyproxy.io/docs/envoy/v1.38.3/configuration/advanced/substitution_formatter#config-advanced-substitution-operators

### 配置输出日志
```
envoy -c envoy-demo.yaml --log-path logs/custom.log //设置日志输出文件
envoy -c envoy-demo.yaml -l debug --log-path logs/custom.log //设置日志级别
```