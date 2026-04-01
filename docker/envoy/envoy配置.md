# envoy 配置
{docsify-updated}

> 配置参考：https://www.envoyproxy.io/docs/envoy/latest/configuration/configuration

## 配置生成器
Envoy 配置会变得相对复杂。在 Lyft 我们使用 jinja 模板使配置变得更加易于创建和管理。发行版源代码包含一个配置生成器的版本，它与我们在 Lyft 中使用的配置生成器近似一致。我们也为上述的三个场景中的每一个都包含了示例配置模板。

+ 生成器脚本： configs/configgen.py
+ 服务对服务模板： configs/envoy_service_to_service_v2.template.yaml
+ 前置代理模板： configs/envoy_front_proxy_v2.template.yaml
+ 双重代理模板： configs/envoy_double_proxy_v2.template.yaml

从仓库的根目录运行以下命令生成示例配置：
```
mkdir -p generated/configs
bazel build //configs:example_configs
tar xvf $PWD/bazel-out/k8-fastbuild/bin/configs/example_configs.tar -C generated/configs
```

## 配置方式
Envoy 支持多种配置方式：
+ 静态文件配置：将配置写在配置文件中，启动时使用指定配置文件
+ 从文件系统动态加载配置文件：与静态文件的差别是能动态加载配置文件中的配置
+ 从控制平面配置（xDs）


### 静态文件配置
要使用静态配置启动 Envoy，需要将 `listeners` 和 `clusters ` 指定为 `static_resources` 。

如果你想使用 admin 管理接口，还可以在配置文件中添加 admin 部分。

```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9999 }

static_resources:

  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  host_rewrite_literal: www.envoyproxy.io
                  cluster: service_envoyproxy_io

  clusters:
  - name: service_envoyproxy_io
    type: LOGICAL_DNS
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: service_envoyproxy_io
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: www.envoyproxy.io
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: www.envoyproxy.io
```

### 基于文件系统的动态配置
我们也可以使用实现了 xDS 协议的文件，以动态配置方式启动 Envoy。**当文件系统中的文件发生变化时，Envoy 会自动更新配置**。请注意：Envoy 只在文件移动替换配置文件时进行更新，而不是在原地编辑文件时进行更新。这样做是为了确保配置的一致性。

使用这种方式，启动 Envoy 时至少需要配置以下部分：
+ 配置 `node` ，以唯一标识代理节点。
+ 配置 `dynamic_resources` 来告诉 Envoy 动态配置的位置。

通常我们至少需要两个动态配置文件：
+ 用于配置 `listener` 的 `lds.yaml` - Listener discovery service (LDS)。
+ 用于配置 `cluster` 的 `cds.yaml`  - Cluster discovery service (CDS)。

如果希望监控 Envoy 或检索统计或配置信息，还可以添加一个管理部分。


示例：
```
node:
  cluster: test-cluster
  id: test-id

dynamic_resources:
  cds_config:
    path: /var/lib/envoy/cds.yaml
  lds_config:
    path: /var/lib/envoy/lds.yaml

admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 19000
```

lds.yaml
```
resources:
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
  name: listener_0
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
  filter_chains:
  - filters:
    - name: envoy.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress_http
        http_filters:
        - name: envoy.router
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
        route_config:
          name: local_route
          virtual_hosts:
          - name: local_service
            domains:
            - "*"
            routes:
            - match:
                prefix: "/"
              route:
                host_rewrite_literal: www.envoyproxy.io
                cluster: example_proxy_cluster
```

cds.yaml:
```
resources:
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: example_proxy_cluster
  type: STRICT_DNS
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http2_protocol_options: {}
  load_assignment:
    cluster_name: example_proxy_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: www.envoyproxy.io
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: www.envoyproxy.io
```