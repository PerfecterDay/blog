# Envoy 健康检查
{docsify-updated}

> https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking#arch-overview-health-checking

Envoy可以为每个不同的上游集群分别配置不同主动健康检查策略。主动健康检查和 EDS 服务发现类型是相辅相成的。不过，在其他情况下，即使使用其他服务发现类型，也需要进行主动健康检查。Envoy 支持几种不同类型的健康检查和各种设置（检查时间间隔、标记主机不健康前所需的失败次数、标记主机健康前所需的成功次数等）：
1. HTTP：在 HTTP 健康检查期间，Envoy 会向上游主机发送 HTTP 请求。默认情况下，如果主机是健康的，它就会得到 200 的响应。预期响应代码和重试响应代码是可配置的。如果上游主机希望立即通知下游主机不再向其转发流量，它可以返回非预期或非可恢复状态代码（默认为任何非 200 代码）。
2. gRPC： 在 gRPC 健康检查期间，Envoy 会向上游主机发送 gRPC 请求。默认情况下，如果主机是健康的，它希望得到 200 的响应。
3. L3/L4：在 L3/L4 健康检查期间，Envoy 会向上游主机发送一个可配置的字节缓冲区。如果主机被认为是健康的，它就会在响应中回传该字节缓冲区。Envoy 还支持仅连接的 L3/L4 健康检查。
4. Redis： Envoy 会发送一条 Redis PING 命令，并期待得到 PONG 响应。上游 Redis 服务器可能会以 PONG 以外的任何其他方式回应，从而导致主动健康检查立即失败。作为选项，Envoy 可以对用户指定的密钥执行 EXISTS。如果键不存在，则视为健康检查通过。这样，用户就可以将指定的键值设置为任意值，然后等待流量耗尽，从而标记 Redis 实例进行维护。
5. Thrift： Envoy 会发送 Thrift 请求，并期待成功响应。上游主机也可能响应异常，导致健康检查失败。

Envoy 健康检查的默认行为：
+ 当该实例健康检查失败次数达到 `unhealthy_threshold`，Envoy 会将其标记为 unhealthy；
+ Envoy 会将其从负载均衡池中移除；
+ 不再向其转发流量。但是在某些特殊情况下（比如集群仅 1 个节点时），envoy 仍然会转发流量给不健康的节点。

默认情况下，Envoy 会启用 “Healthy Panic Mode”，也就是：如果健康主机比例低于 `healthy_panic_threshold` （默认是 50%），**Envoy 会忽略健康检查结果，继续向所有主机转发流量，避免“完全不可用”**。
所以：
+ 如果只有 1 台主机且它 unhealthy ⇒ 健康比例 0%；
+ 小于默认的 50% ⇒ panic 模式触发 ⇒ 仍然会继续向该主机转发流量！

也就是只有健康比例大于等于这个阈值配置时，才会停止流量转发。所以，如果想要即使只有一个节点当发现不健康时也不转发流量，就需要讲这个阈值设置为0：
```
common_lb_config:
      healthy_panic_threshold:
        value: 0
```

## 主动健康检查
Envoy 作为健康检查的发起方主动向上游集群发起健康检查请求，并根据响应判断上游节点的健康状态。

### 主动健康检查的配置
Envoy 默认不会自动开启健康检查，必须显式在 cluster 配置中添加 `health_checks` 字段。，Envoy 才会对上游服务进行健康探测。示例：
```
clusters:
- name: my_service
  connect_timeout: 0.25s
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
      cluster_name: search-cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            health_check_config:
              port_value: 8091
              address:
                socket_address:
                  address: search-service-svc
                  port_value: 8091
            address:
              socket_address:
                address: search-service-svc
                port_value: 8000
                ipv4_compat: true
  common_lb_config:
      healthy_panic_threshold:
        value: 0
  health_checks:
  - timeout: 1s
    interval: 5s
    unhealthy_threshold: 3
    healthy_threshold: 2
    http_health_check:
      path: /health
      codec_client_type: HTTP1
      expected_statuses:
      - start: 200
        end: 201    # 只接受 200，[start,end)
      - start: 204
        end: 205    # 只接受 204
```


| 字段                   | 说明                          |
|------------------------|-------------------------------|
| `timeout`              | 单次健康检查超时时间          |
| `interval`             | 健康检查间隔                  |
| `unhealthy_threshold`  | 连续失败多少次视为不健康      |
| `healthy_threshold`    | 连续成功多少次视为恢复健康    |
| `http_health_check.path` | 发起 HTTP 请求的路径       |
| `http_health_check.expected_statuses` | 标记健康检查成功的响应码   |
| `tcp_health_check`     | TCP 层的健康检查方式（如不用 HTTP） |
| `grpc_health_check`    | GRPC 方式健康检查             |



### 主动健康检查的日志
默认日志行为：
1. 健康检查成功：
    + 不会每次都记录日志；
    + 只在主机从“不健康”恢复为“健康”状态时（即达到 healthy_threshold）打印一条恢复日志。
2. 健康检查失败：
    + 只在连续失败达到 `unhealthy_threshold` ，主机状态从“健康”变为“不健康”时，打印一条日志；
    + 除非开启了 `always_log_health_check_failures` ，否则中间的失败也不会被记录。

中间的失败/成功都不会单独打印日志。

```
{"health_checker_type":"HTTP","host":{"socket_address":{"protocol":"TCP","address":"192.168.168.129","resolver_name":"","ipv4_compat":false,"port_value":8000}},"cluster_name":"search-cluster","add_healthy_event":{"first_check":true},"timestamp":"2025-07-02T07:55:38.260Z"} --- 健康检查成功的日志

{"health_checker_type":"HTTP","host":{"socket_address":{"protocol":"TCP","address":"192.168.168.129","resolver_name":"","ipv4_compat":false,"port_value":8000}},"cluster_name":"search-cluster","health_check_failure_event":{"failure_type":"ACTIVE","first_check":true},"timestamp":"2025-07-02T06:52:45.039Z"} --- 健康检查失败的日志

{"health_checker_type":"HTTP","host":{"socket_address":{"protocol":"TCP","address":"192.168.168.129","resolver_name":"","ipv4_compat":false,"port_value":8000}},"cluster_name":"search-cluster","eject_unhealthy_event":{"failure_type":"NETWORK"},"timestamp":"2025-07-02T07:58:09.811Z"} --- 健康检查失败的日志
```

### 查看集群的健康状态
通过 admin API 能查看集群健康信息：
```
# curl localhost:9999/clusters | grep search
search-cluster::192.168.168.129:8000::health_flags::healthy -- 健康

search-cluster::192.168.168.129:8000::health_flags::/failed_active_hc --不健康
```
当集群不是健康状态时，有三种标志位：
+ `/failed_active_hc` ： 主机的主动健康检查失败。
+ `/failed_eds_health` ： 主机被 EDS 标记为不健康。
+ `/failed_outlier_check`： 主机未通过被动离群健康检查。

### HTTP健康检查过滤器
当在集群之间部署具有主动健康检查的 Envoy 服务网格时，服务网格可能会检查服务的状态进而可能会产生大量的健康检查流量。此时，Envoy 作为被健康检查的对象。检查方通常是 k8s 或其他基础设施组件。

Envoy 提供了一个 HTTP 健康检查过滤器，可以安装在配置好的 HTTP listener 中。该过滤器支持多种运行模式：
1. 无透传（No pass through）  
在此模式下，健康检查请求不会被转发到本地服务。  
Envoy 会根据其当前的**“耗尽状态”（draining state）**返回 200 或 503：
+ 如果未处于 draining 状态 ⇒ 返回 200（健康）；
+ 如果处于 draining 状态 ⇒ 返回 503（不健康）。

2. 无透传，基于上游集群健康状态计算  
在此模式下，健康检查过滤器会根据一个或多个上游集群中“可用服务器”（包括 healthy 和 degraded）的比例，来返回 200 或 503。
+ 如果可用服务器比例 ≥ 指定阈值 ⇒ 返回 200；
+ 否则 ⇒ 返回 503；
+ 注意： 如果 Envoy 本身处于 draining 状态，无论上游集群是否健康，都会返回 503。  

3. 透传模式（Pass through）  
在此模式下，Envoy 会将每一个健康检查请求转发给本地服务。
+ 由本地服务根据其健康状态返回 200 或 503；
+ Envoy 只是转发器，不做干预。

4. 透传 + 缓存（Pass through with caching）   
在此模式下：
+ Envoy 会把健康检查请求转发给本地服务；
+ 但将本地服务的响应结果缓存一段时间；
+ 之后的健康检查请求，在缓存时间内会直接返回缓存值；
+ 缓存时间到达后，下一次请求才会再次调用本地服务。

这是在部署大型服务网格时推荐使用的模式。原因是：
1. Envoy 对健康检查请求使用 长连接（persistent connections）；
2. 健康检查请求对 Envoy 来说几乎没有开销；
3. 此模式可以获得一个最终一致的健康视图，而不会因大量探测请求而压垮本地服务。


## 被动健康检查
Envoy 还通过 outlier detection(立群检测)的方式支持被动健康检查。这种模式下， Envoy 会被动监控上游主机的对业务请求响应的响应结果，异常时将主机节点驱逐出集群。比如，某台主机在一段时间内频繁的返回 5xx 的报错， Envoy 就会认为该节点不健康，会讲其踢出集群的负载均衡（panic 模式除外）。

异常点检测和剔除是动态确定上游集群中是否有一些主机的性能与其他主机不同，并将其从健康的负载平衡集中剔除的过程。性能可能沿着不同的轴，如连续故障、时间成功率、时间延迟等。异常值检测是被动健康检查的一种形式。Envoy 还支持主动健康检查。被动健康检查和主动健康检查可以同时启用，也可以单独启用，它们构成了整体上游健康检查解决方案的基础。异常值检测是群集配置的一部分，它需要过滤器来报告错误、超时和重置。目前，以下过滤器支持异常值检测：http 路由器、tcp 代理、redis 代理和 thrift 代理。

检测到的错误分为两类：**外部错误和本地错误**。外部错误针对特定事务，发生在上游服务器上，以响应接收到的请求。例如，HTTP 服务器返回错误代码 500 或 Redis 服务器返回无法解码的有效负载。这些错误是在 Envoy 成功连接到上游主机后产生的。本地错误是 Envoy 与上游主机通信时产生的错误，通常表示无法连接到上游主机。本地错误的例子包括超时、TCP 重置、无法连接到指定端口等。

Envoy 支持通过统计这些错误信息并基于这些信息根据一些算法来决定某个节点是否健康，并且根据健康状态来将节点提出负载均衡的集群以避免流量发送到不正常的节点。

## 快速使健康检查失败
在使用主动健康检查和被动健康检查（离群检测）时，通常会使用较长的健康检查间隔，以避免大量的主动健康检查流量。在一些情况下，比如相对某个节点进行运维升级时，我们想想让 envoy 停止转发流量到特定节点，而不是等到 envoy 下次健康检查时才能检测到。 Envoy 有两种方案支持这一需求：
1. 路由器过滤器（ router filter）和 HTTP 主动健康检查器将检测 `x-envoy-immediate-health-check-fail` 标头。如果上游主机设置了这个头（即使是在普通业务流量而非健康检查的响应中），Envoy 会立即将该主机标记为主动健康检查失败，并将其排除在负载平衡之外。请注意，只有当主机的集群配置了主动健康检查时，才会出现这种情况。
2. 对于对 Envoy 节点本身的健康检查流量，可以通过 `/healthcheck/fail` 管理端点将 Envoy 标记为失败，HTTP健康检查过滤器会自动设置 ``x-envoy-immediate-health-check-fail`` 标头。

如果某个节点通过响应头 `x-envoy-immediate-health-check-fail` 将节点标记为非健康后，想要恢复为健康状态的话，要怎么办呢 ？ 答案是不需要额外操作， envoy 会通过健康检查机制自动检测节点状态，也就是说如果后续 envoy 的健康检查成功次数达到配置的值，就会将节点状态标记为 `healthy` ，路由器会重新给节点转发流量。

这种特性特别适合运维场景，比如我们需要升级某台服务器时，我们可以在服务内置一个运维接口，在这个接口中返回 `x-envoy-immediate-health-check-fail: true` 的响应头，然后通过 envoy 请求这个接口（这里有另外一个问题就是如果集群有多台机器，如何保证通过envoy请求到特定的节点呢？），这样 envoy 就会立刻将这个节点标记为 `/failed_active_hc/excluded_via_immediate_hc_fail` 状态，并且停止流量转发。后续如果运维结束， envoy 会自动通过健康检查机制将节点状态标记为 healthy。

如上所述，我们遇到的一个问题是无法通过 envoy 调用特定节点的接口，那么我们可以在服务节点内部增加一个设置开关，当开关打开时，服务节点的任意请求会返回 `x-envoy-immediate-health-check-fail` 这通过 filter/interceptor 很容易实现。当运维时打开这个开关（可以通过 curl 脚本/配置文件等），这样也能使节点快速失败。