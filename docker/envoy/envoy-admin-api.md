# envoy web 管理 API
{docsify-updated}

> https://www.envoyproxy.io/docs/envoy/v1.18.4/operations/admin

开启管理员后台：
```
admin:
  profile_path: /tmp/envoy.prof
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }
```

## GET /
访问控制台主页

## GET /help
帮助页面

## GET /certs
查看所有的TLS证书信息

## GET /clusters
查看所有的上游集群信息
| Name             | Type    | Description                                                                 |
|------------------|---------|-----------------------------------------------------------------------------|
| cx_total         | Counter | Total connections                                                           |
| cx_active        | Gauge   | Total active connections                                                    |
| cx_connect_fail  | Counter | Total connection failures                                                   |
| rq_total         | Counter | Total requests                                                              |
| rq_timeout       | Counter | Total timed out requests                                                    |
| rq_success       | Counter | Total requests with non-5xx responses                                       |
| rq_error         | Counter | Total requests with 5xx responses                                           |
| rq_active        | Gauge   | Total active requests                                                       |
| healthy          | String  | The health status of the host. See below                                   |
| weight           | Integer | Load balancing weight (1-100)                                               |
| zone             | String  | Service zone                                                                |
| canary           | Boolean | Whether the host is a canary                                                |
| success_rate     | Double  | Request success rate (0-100). -1 if not enough [request volume](#) in interval |

`GET /clusters?format=json` json 格式输出


## GET /config_dump
Dump 当前的 envoy 配置

## POST /cpuprofiler
启用或禁用 CPU 剖析器。需要使用 gperftools 进行编译。输出文件可通过 `admin.profile_path` 配置。