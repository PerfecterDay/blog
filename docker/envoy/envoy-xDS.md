# envoy-xDS 实战
{docsify-updated}

> https://cloudnative.to/envoy/configuration/overview/xds_api.html#rest

+ EDS(Endpoint Discovery Service API) 提供了一种更先进的机制，使 Envoy 可以发现上游集群的成员。
+ CDS(Cluster Discovery Service API) 发现上游集群
+ RDS(Route Discovery Service API) Envoy 可以在运行时发现 HTTP 连接管理器的整个路由配置。
+ VHDS(Virtual Host Discovery Service)虚拟主机发现服务允许根据需要请求属于路由配置的虚拟主机，并将其与路由配置本身分开
+ SRDS(Scoped Route Discovery Service) 允许将路由表分割成多个部分
+ LDS(Listener Discovery Service) 建立了一种机制，Envoy 可通过这种机制在运行时发现监听器配置
+ SDS(Secret Discovery Service) 允许 Envoy 在运行时发现 TLS 上下文中的加密设置
+ RTDS(Runtime Discovery Service) 允许 Envoy 在运行时发现runtime配置
+ ECDS()
+ Aggregated xDS ("ADS")

基于 envoy-1.31.0 ：
1. [clone repo](https://github.com/envoyproxy/java-control-plane)
2. 编译运行 repo 中的 TestMain
3. 使用下述 envoy 配置文件启动 envoy(或者参照repo src/test/resources/envoy/xds.v3.config.yaml 文件)：
    ```
    admin:
    address:
        socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 9901
    static_resources:
    listeners:
    - name: listener_0
        address:
        socket_address:
            protocol: TCP
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
            route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                domains: ["*"]
                routes:
                # - match:
                #     prefix: "/"
                #   route:
                #     host_rewrite_literal: www.envoyproxy.io
                #     cluster: service_envoyproxy_io
            http_filters:
            - name: envoy.filters.http.router
                typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
    clusters:
    - name: xds_cluster
        type: STRICT_DNS
        # Comment out the following line to test on v6 networks
        dns_lookup_family: V4_ONLY
        lb_policy: ROUND_ROBIN
        load_assignment:
        cluster_name: xds_cluster
        endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                socket_address:
                    address: localhost
                    port_value: 12345
        http2_protocol_options: {}
        # transport_socket:
        #   name: envoy.transport_sockets.tls
        #   typed_config:
        #     "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        #     sni: www.envoyproxy.io
    node:
    cluster: test-cluster
    id: test-id

    dynamic_resources:
    cds_config:
        api_config_source:
        api_type: GRPC
        grpc_services:
            envoy_grpc:
            cluster_name: xds_cluster
        transport_api_version: V3
        resource_api_version: V3
    lds_config:
        api_config_source:
        api_type: GRPC
        grpc_services:
            envoy_grpc:
            cluster_name: xds_cluster
        transport_api_version: V3
        resource_api_version: V3
    ```
   xDS 服务集群要在 static_resources 下配置，并且要配置禁用 TLS。
