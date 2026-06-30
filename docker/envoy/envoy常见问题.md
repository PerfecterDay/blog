# envoy 常见配置问题

+ uatfile upload host 不匹配返回403 -> 需要启用host rewrite `auto_host_rewrite`
   `route: { cluster: uatfile-cluster, timeout: { seconds: 10 } ,auto_host_rewrite: true}`
+ 设置超时时间: timeout
   `route: { cluster: uatfile-cluster, timeout: { seconds: 60 } ,auto_host_rewrite: true}`
+ 上游服务配置 https ----> `transport_socket` https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/securing
   ```
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
   ```
+ uatfile 返回413,Payload Too Large ---> 配置 `per_connection_buffer_limit_bytes` https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/flow_control#faq-flow-control
   ```
   clusters:
    name: cluster_0
    connect_timeout: 5s
    per_connection_buffer_limit_bytes: 16000000
    load_assignment:
      cluster_name: some_service
      endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: ::1
                  port_value: 46685
   ```

+ 正则表达式超长报错---> 需要修改正则表达式长度
regex '^/user/(488/sms|h5/getJwt|captcha|version/getAIDrawTemplate|version/isMainLand|version/promotionTemplate|version/activityTemplate2).*' RE2 program size of 106 > max program size of 100 set for the error level threshold. Increase configured max program size if necessary.

+ x-forwarded-for 失效
    Envoy will only append to XFF if the use_remote_address HTTP connection manager option is set to true and the skip_xff_append is set false. This means that if use_remote_address is false (which is the default) or skip_xff_append is true, the connection manager operates in a transparent mode where it does not modify XFF.

+ x-envoy-external-address: 追踪原始客户端IP
	It is a common case where a service wants to perform analytics based on the origin client’s IP address. Per the lengthy discussion on XFF, this can get quite complicated, so Envoy simplifies this by setting x-envoy-external-address to the trusted client address if the request is from an external client. x-envoy-external-address is not set or overwritten for internal requests. This header can be safely forwarded between internal services for analytics purposes without having to deal with the complexities of XFF.

+ 容器运行权限
```
sudo docker run -idt --name envoy --restart=always -e TZ="Asia/Shanghai" -e loglevel=debug -v /etc/localtime:/etc/localtime -v "/root/dev/envoy/logs:/tmp" -v "/root/dev/envoy/test.cer:/etc/envoy/test.cer" -v "/root/dev/envoy/testserver.key:/etc/envoy/test.key" -v "/root/dev/envoy/envoy.yaml:/etc/envoy/envoy.yaml" --net=host  4b976b6b0e19
```

默认情况下，Docker 镜像将以构建时创建的 envoy 用户来运行。 envoy uid 和 gid 的默认值都是 101 。
envoy 用户的 uid 和 gid 可以在运行时使用 ENVOY_UID 和 ENVOY_GID 这两个环境变量来设定。这也可以在 Docker 命令行中来完成设定
`$ docker run -d --name envoy -e ENVOY_UID=777 -e ENVOY_GID=777 -p 9901:9901 -p 10000:10000 envoy:v1`

如果你把应用程序日志、管理和访问日志输出到一个文件，envoy 用户将需要足够的权限来写这个文件。这个可以通过设置 ENVOY_UID 或者通过将文件变成 envoy 用户可写的方法来实现。
```
mkdir logs
chown 777 logs
docker run -d -v `pwd`/logs:/var/log --name envoy -e ENVOY_UID=777 -p 9901:9901 -p 10000:10000 envoy:v1
```
随后，你可以配置 envoy 将日志文件输出在 /var/log 文件里。

有一种可以不用改变任何文件的权限或者在容器内部使用 root 用户的方法就是在启动容器的时候使用宿主机用户的 uid :
`docker run -d --name envoy -e ENVOY_UID=`id -u` -p 9901:9901 -p 10000:10000 envoy:v1`



+ `-e loglevel=debug`: 设置日志级别
logs 文件夹报 permission denied 错误 -> 必须进入容器内部后使用 `chown envoy /tmp` 改变目录所有者，然后运行容器  
(access_log)[https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log]


+ 路由正则太复杂也会报错：`envoy RE2 program size of 117 > max program size of 100 set for the error level threshold. Increase configured max program size if necessary.` ,修复方案就是将路由拆分成多条
  ```
  safe_regex: { google_re2: {}, regex: "^/user/(488/sms|captcha|version/getAIDrawTemplate|version/isMainLand|version/promotionTemplate|version/activityTemplate2).*" }
  safe_regex: { google_re2: {}, regex: "^/user/(488/sms|h5/getJwt|captcha|version/.*Template.*|version/isMainLand|version/getSensorsSource).*" }
  ```

## RESPONSE_CODE返回0
当 Envoy 代理在访问日志中返回 %RESPONSE_CODE% = 0 时，通常意味着 Envoy 未能从上游服务收到任何有效的 HTTP 响应码，或者连接在响应生成前就已经中断。发生此问题通常由以下几种常见原因导致：
1. 客户端提前关闭连接这是最常见的原因。在客户端（例如移动网络或不稳定的网络环境）发起请求后，可能由于超时、用户主动取消或网络抖动，在 Envoy 将请求发送给上游服务并收到回复之前，客户端就已单方面中断了连接。此时 Envoy 记录的响应码即为 0。
2. 上游连接重置或中断 (Connection Reset/Drop)如果 Envoy 成功将请求转发给了上游服务，但在上游服务返回响应的头部（Header）之前，与上游的 TCP 连接发生了中断（例如上游服务崩溃、Pod 被杀、或网络策略阻断），Envoy 无法读取到任何响应码，从而记录为 0。
3. 请求超时 (Timeout)如果在路由中配置了严格的超时时间，而上游服务处理缓慢未能在限制时间内返回数据，Envoy 会主动中断转发。如果此时响应头还未生成，日志中的状态码就会显示为 0。
4. Envoy 内部处理错误Envoy 本身在处理请求时发生严重的内部错误（例如解析 HTTP 协议失败、路由匹配完全失效且未返回默认错误等），也会直接拦截请求并丢弃，无法输出标准响应码。