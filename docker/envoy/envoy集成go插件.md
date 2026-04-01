# 开发 Go 插件
{docsify-updated}
> https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/golang-http.html

- [开发 Go 插件](#开发-go-插件)
  - [StreamFilter 接口的调用顺序](#streamfilter-接口的调用顺序)
  - [配置插件](#配置插件)
  - [测试 curl](#测试-curl)

```
docker compose -f docker-compose-go.yaml run --rm go_plugin_compile
docker compose up --build -d
```

### StreamFilter 接口的调用顺序
```
DecodeHeaders -> DecodeData -> DecodeData -> EncodeHeaders -> EncodeData -> EncodeData -> OnLog -> OnDestroy
```

### 配置插件
```
- filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          http_filters:
          - name: envoy.filters.http.golang
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.golang.v3alpha.Config
              library_id: simple
              library_path: "lib/simple.so"
              plugin_name: simple
              plugin_config:
                "@type": type.googleapis.com/xds.type.v3.TypedStruct
                value:
                  prefix_localreply_body: "Configured local reply from go"
```

### 测试 curl
```
curl -v localhost:10000/update_upstream_response -H 'token: 123' -XPOST -d '{scs}'
```