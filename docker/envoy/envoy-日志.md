# envoy 日志
{docsify-updated}




## Access Log
> https://cloudnative.to/envoy/configuration/observability/access_log/usage.html#config-access-log-format-dictionaries

1. `%RESPONSE_FLAGS%`
如果有的话，表示响应或者连接的附加详情。 对于 TCP 连接，说明中提到的响应码不适用。 可能的值如下：

HTTP 和 TCP
+ UH: 附加在 503 响应状态码，表示上游集群中无健康的上游主机。
+ UF: 附加在 503 响应状态码，表示上游连接失败
+ UO: 附加在 503 响应状态码，表示上游溢出 (熔断) 。
+ NR: 附加在 404 响应状态码，表示无给定请求的 路由配置 ，或者对于一个下游连接没有匹配的过滤器链。
+ URX: 请求因为达到了上游重试限制 (HTTP) 或者最大连接尝试 (TCP) 而被拒绝。

HTTP 独有
+ DC: 下游连接中断。
+ LH: 附加在 503 响应状态码，本地服务 健康检查请求 失败。
+ UT: 附加在 504 响应状态码，上游请求超时。
+ LR: 附加在 503 响应状态码，连接在本地被重置。
+ UR: 附加在 503 响应状态码，连接在远程被重置。
+ UC: 附加在 503 响应状态码，上游连接中断。
+ DI: 通过 故障注入 使请求处理被延迟一个指定的时间。
+ FI: 通过 故障注入 使请求被终止掉并带有一个相应码。
+ RL: 附加在 429 相应状态码，请求被 HTTP 限流过滤器 本地限流。
+ UAEX: 请求被外部授权服务拒绝。
+ RLSE: 因限流服务出现错误而拒绝请求。
+ IH: 附加在 400 响应状态码，请求被拒绝因为他为 strictly-checked header 设置了一个无效值。
+ SI: 附加在 408 相应状态码，流闲置超时。
+ DPE: 下游请求存在一个 HTTP 协议错误。
+ UMSDR: 上游请求达到最大流持续时长

### 配置输出日志
envoy -c envoy-demo.yaml --log-path logs/custom.log