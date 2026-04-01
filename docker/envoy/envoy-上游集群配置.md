# envoy 上游集群配置
{docsify-updated}

> https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery-types-static


## 服务发现


## 配置TLS
可以分别配置 `DownstreamTlsContext` 和 `UpstreamTlsContext`。

```
static_resources:
  listeners:
  - name: listener_0
    address: {socket_address: {address: 127.0.0.1, port_value: 10000}}
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            virtual_hosts:
            - name: default
              domains: ["*"]
              routes:
              - match: {prefix: "/"}
                route:
                  cluster: some_service
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          allow_renegotiation: true
          common_tls_context:
            tls_certificates:
            - certificate_chain: {filename: "certs/servercert.pem"}
              private_key: {filename: "certs/serverkey.pem"}
            validation_context:
              trusted_ca:
                filename: certs/cacert.pem
  clusters:
  - name: some_service
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: some_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1234
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        allow_renegotiation: true
        common_tls_context:
          tls_certificates:
          - certificate_chain: {"filename": "certs/servercert.pem"}
            private_key: {"filename": "certs/serverkey.pem"}
            ocsp_staple: {"filename": "certs/server_ocsp_resp.der"}
          validation_context:
            match_typed_subject_alt_names:
            - san_type: DNS
              matcher:
                exact: "foo"
            trusted_ca:
              filename: /etc/ssl/certs/ca-certificates.crt
```