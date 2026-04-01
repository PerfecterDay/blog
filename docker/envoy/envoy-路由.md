# envoy 路由
{docsify-updated}

> https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing.html


Envoy 内置了一个 HTTP Router Filter（路由过滤器），用于在边缘代理或服务网格中根据配置将请求转发到对应的上游集群（Upstream Cluster）。

它还支持作为正向代理（forward proxy），客户端可将 Envoy 设置为 HTTP 代理使用。

路由工作流程:
1. Envoy 接收一个 HTTP 请求。
2. 匹配请求对应的上游集群。
3. 从连接池获取一个连接到该集群的宿主机（host）。
4. 转发请求至宿主机。

## 路由过滤器支持的功能
1. 虚拟主机（Virtual Hosts）与集群映射
2. 根据请求路径、前缀、请求头匹配
3. 重写路径、前缀或主机
4. 请求重定向
5. 超时设置、重试与 Hedging（多请求竞态）
6. 流量分流与切换
7. 基于策略的路由转发
8. 直接响应（Direct Response）
9. 对于非 TLS 的代理，支持绝对 URL 转发


### 直接响应
Envoy 支持发送 "直接 "响应。这些是预配置的 HTTP 响应，不需要转发请求到代理的上游服务器。

在路由中指定直接回复有两种方法：
+ 设置 `direct_response` 字段。这适用于所有 HTTP 响应状态。
+ 设置 `redirect` 字段。这仅适用于重定向响应状态，但可简化 `Location` 标头的设置。

```
direct_response的定义：
{
  "status": ...,
  "body": {...}
}

body的定义如下：
{
  "filename": ...,
  "inline_bytes": ...,
  "inline_string": ...,
  "environment_variable": ...,
  "watched_directory": {...}
}
```

示例配置：
```
- match: {prefix: "/"}
  direct_response:
    status: 503
    body:
      inline_string: '{ "status" : "E0009", "message": "Service Unavailable" }'
```

### 路由重写与添加请求头
重写请求路径/对特定路由禁用三防插件/转发时增加请求头
```
- match:
    prefix: "/h5/fund"
typed_per_filter_config:
    envoy.filters.http.ext_authz:
    "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthzPerRoute
    disabled: true
route: { cluster: financial-management-cluster, timeout: { seconds: 10 } , regex_rewrite: {pattern: {regex: "/h5/fund", google_re2: {}},substitution: "/fund"} }
- match:
    safe_regex: { google_re2: {}, regex: "^/[service|yichat].*" }
route: { cluster: kefu-cluster, timeout: { seconds: 10 } }
- match:
    safe_regex: { google_re2: {}, regex: "^/gtjaiback.*" }
  route: { cluster: bpm-cluster, timeout: { seconds: 10 } }
  request_headers_to_add: 
  - header:
      key: "appId"
      value: "1e6c3a95-552a-5d04-38ad-2f4e238236b6"
```

### 配置websocket
 ```
 - match:
      safe_regex: { google_re2: {}, regex: "^/yichat.*" }
  typed_per_filter_config:
      envoy.filters.http.ext_authz:
      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthzPerRoute
      disabled: true
  route: { cluster: kefu-cluster-1, timeout: { seconds: 0 }, upgrade_configs: [{upgrade_type: websocket}] }
 ```