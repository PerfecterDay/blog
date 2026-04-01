# envoy 熔断
{docsify-updated}

> https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking

Circuit Breaker 触发后，Envoy 会立即返回 本地响应（通常是 503），不再尝试连接上游主机。

## 熔断配置选项
1. 集群最大连接数(max_connections)   
Envoy 与上游群集中所有主机建立连接的最大数量。如果超过最大连接数，群集的 `upstream_cx_overflow` 计数器将递增。所有连接，无论是活动连接还是耗尽连接，都将计入最大连接数计数中。即使触发可这个断路器，Envoy 也会确保群集负载平衡选择的主机至少分配有一个连接。这意味着 `upstream_cx_active` 数量可能高于群集最大连接数断路器，其上限为 $cluster maximum connections + (number of endpoints in a cluster) * (connection pools for the cluster)$ 。该上限适用于所有工作者线程的连接数总和。

2. 群集最大待处理请求数(max_pending_requests)
等待连接池连接时排队等待的最大请求数。只要没有足够的上游连接来立即分派请求，请求就会被添加到待处理请求列表中。对于 HTTP/2 连接，如果未配置最大并发流和每个连接的最大请求数，所有请求都将在同一连接上多路复用，因此只有在未建立连接时才会触发此断路器。如果断路器溢出，群集的 `upstream_rq_pending_overflow` 计数器将递增。对于 HTTP/3，与 HTTP/2 的最大并发流相当的是最大并发流

3. 群集最大请求数(max_requests)
在任何给定时间内，群集中所有主机未处理的最大请求数。如果该断路器溢出，群集的 `upstream_rq_pending_overflow` 计数器将递增。

4. 群集最大重试次数(max_retries)  
在任何给定时间内，群集中所有主机未完成的重试次数上限。一般情况下，我们建议使用重试预算；但如果首选静态断路，则应积极进行断路重试。这样做的目的是允许对零星故障进行重试，但总的重试次数不能过多，以免造成大规模级联故障。如果断路器溢出，群集的 `upstream_rq_retry_overflow` 计数器将递增。

5. 集群最大并发连接池   
可并发实例化的最大连接池数量。某些功能（如原始源监听器过滤器）可创建数量不限的连接池。当集群用完并发连接池后，会尝试回收一个空闲的连接池。如果无法回收，断路器就会溢出。这与群集最大连接数不同，连接池不会超时，而连接通常会超时。连接会自动清理，而连接池不会。需要注意的是，连接池要发挥作用，至少需要一个上游连接，因此该值不应大于群集最大连接数。如果断路器溢出，群集的 `upstream_cx_pool_overflow` 计数器将递增。

每个断路限制可根据每个上游群集和每个优先级进行配置和跟踪。这使得分布式系统的不同组件可以独立调整，并具有不同的限制。这些断路器的实时状态，包括断路器打开前剩余的资源数量，都可以通过统计数据观察到。

工作线程共享断路器限制，例如，如果活动连接阈值为 500，工作线程 1 有 498 个活动连接，那么工作线程 2 只能再分配 2 个连接。由于执行最终是一致的，因此线程之间的竞争可能会导致超出限制。

请注意，如果是 HTTP 请求，断路会导致路由器过滤器设置 `x-envoy-overloaded` 标头。

## 关闭熔断
断路器默认启用，默认值适中，例如每个群集 1024 个连接。要禁用断路器，请将阈值设置为允许的最高值。
```
circuit_breakers:
  thresholds:
    - priority: DEFAULT
      max_connections: 1000000000
      max_pending_requests: 1000000000
      max_requests: 1000000000
      max_retries: 1000000000
    - priority: HIGH
      max_connections: 1000000000
      max_pending_requests: 1000000000
      max_requests: 1000000000
      max_retries: 1000000000
```

## 自定义熔断返回
```
http_filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    stat_prefix: ingress_http
    route_config: {...}
    http_filters:
    - name: envoy.filters.http.router
    local_reply_config:
      mappers:
        - filter:
            response_flag_filter:
              flags:
                - UF  # Upstream Overflow = 熔断触发（请求数满）
          status_code: 529   # 自定义 HTTP 状态码（例如非标准码）
          body_format_override:
            text_format: '{"code":529, "message":"熔断器已触发，服务繁忙"}'
```

`response_flag_filter.flags`:
+ UF（Upstream Overflow）：表示熔断器触发，Envoy 拒绝请求。
+ NR：No Route
+ UH：No healthy upstream
+ UT：Upstream timeout

`status_code`: 设置为你想要返回的 HTTP 状态码，如：
+ 标准码如 429（Too Many Requests）
+ 自定义码如 529（注意客户端可能不识别）

`body_format_override` : 自定义响应体，可以返回 JSON、文本等。