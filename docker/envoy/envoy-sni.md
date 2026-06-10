# Envoy 上游 TLS 握手失败排查：`SSLV3_ALERT_HANDSHAKE_FAILURE`

## 现象

curl 请求 `GET /tcapi/abc` 经 Envoy 转发后返回 `503`，Envoy debug 日志关键片段：

```
[C3] connecting to 3.169.183.106:443
[C3] connected
[C3] TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE
              268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO
[C3] client disconnected, failure reason: TLS error ...
upstream reset: reset reason: connection failure,
  transport failure reason: TLS error: ... HANDSHAKE_FAILURE_ON_CLIENT_HELLO
```

## 直接结论

**Envoy 在向上游 `api.tradingcentral.com:443` 建立 TLS 连接时没有发送 SNI（Server Name Indication）**，上游服务器（IP `3.169.183.106` 属 AWS 段，目标域名大概率挂在共享 CDN/ALB 后面）因此无法决定返回哪张证书，直接用 `handshake_failure` alert 拒绝了 ClientHello。

## 证据链

### 1. 报错语义

- `HANDSHAKE_FAILURE_ON_CLIENT_HELLO` 是 BoringSSL/Envoy 客户端**收到 server 端发来的 alert 后**抛出的错误。
- TCP 三次握手成功（日志里有 `[C3] connected`），网络通、IP 通、443 端口开放。
- 失败发生在 TLS 第一步——server 解析完 Envoy 的 ClientHello 就拒绝了。

### 2. `tc-trade-cluster` TLS 配置缺少 `sni`

`envoy.yaml`：

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    common_tls_context:
      validation_context:
        trusted_ca:
          filename: /etc/ssl/certs/ca-certificates.crt
```

`UpstreamTlsContext` 里**没有 `sni` 字段**，也没开 `auto_sni`。Envoy 1.18 的默认行为：未显式配置 `sni` 时，TLS ClientHello 里就**不带 SNI 扩展**。对挂 CDN/ALB 的 HTTPS 服务而言，这等同于"匿名敲门"，server 端必然拒绝。

### 3. `:authority` 是 `localhost:7000` 帮不上忙

请求头里 `:authority = localhost:7000`（curl 发的），即使开了 `auto_sni`，它也只会用这个作为 SNI——`localhost` 这个 SNI 上游同样不认。

## 验证方法（不改配置先确认）

在能访问 envoy 出网的机器上：

```bash
# 1) 不带 SNI 握手 —— 应复现 handshake_failure
openssl s_client -connect api.tradingcentral.com:443 -servername "" </dev/null

# 2) 带 SNI 握手 —— 应正常拿到证书
openssl s_client -connect api.tradingcentral.com:443 \
                 -servername api.tradingcentral.com </dev/null
```

第 1 条复现报错、第 2 条成功，即 100% 确认是 SNI 问题。

## 修复方法

在 `tc-trade-cluster` 的 `transport_socket.typed_config` 里加 `sni`：

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    sni: api.tradingcentral.com          # 新增
    common_tls_context:
      validation_context:
        match_subject_alt_names:         # 顺手补证书域名校验
          - exact: api.tradingcentral.com
        trusted_ca:
          filename: /etc/ssl/certs/ca-certificates.crt
```

要点：

- `sni` 写**域名字符串**（不带端口、不带 scheme）。
- 建议同时加 `match_subject_alt_names`，否则当前配置只校验 CA 链、不校验域名，存在中间人风险。
- 若整批 cluster 都缺 SNI，可考虑 `auto_sni: true` + `auto_san_validation: true`（在 `upstream_http_protocol_options` 里），用客户端的 `Host` 头当 SNI；但前提是客户端送的 Host 正确，curl 带 `localhost:7000` 这种就不行。

## 与 `cert.pem` 问题的关系

- 本错误**不是 CA 信任问题**——CA 不被信任会报 `CERTIFICATE_VERIFY_FAILED`，不是 `SSLV3_ALERT_HANDSHAKE_FAILURE`。
- 即使把 `cert.pem` 挂载到 `/etc/ssl/certs/ca-certificates.crt` 并被引用，也救不了这个 503——根本走不到证书校验那一步，server 在 ClientHello 阶段就把连接断了。
- 两个问题相互独立，先解决 SNI 即可恢复 `/tcapi/*` 的访问。

## 排查 Checklist

- [ ] 用 `openssl s_client` 带/不带 `-servername` 复现
- [ ] 给 `tc-trade-cluster` 加 `sni: api.tradingcentral.com`
- [ ] 检查 envoy.yaml 中其余走 TLS 上游的 cluster 是否同样缺 SNI
- [ ] 加 `match_subject_alt_names` 强化证书域名校验
- [ ] 重启 envoy / 重新部署后用 curl 复测 `/tcapi/abc`

## 报错日志
```
[2026-06-09 10:03:20.563][16][debug][conn_handler] [source/server/active_tcp_listener.cc:328] [C2] new connection
[2026-06-09 10:03:20.564][16][debug][http] [source/common/http/conn_manager_impl.cc:261] [C2] new stream
[2026-06-09 10:03:20.565][16][debug][http] [source/common/http/conn_manager_impl.cc:882] [C2][S5882164773424644491] request headers complete (end_stream=true):
':authority', 'localhost:7000'
':path', '/tcapi/abc'
':method', 'GET'
'user-agent', 'curl/8.7.1'
'accept', '*/*'

[2026-06-09 10:03:20.565][16][debug][http] [source/common/http/filter_manager.cc:779] [C2][S5882164773424644491] request end stream
[2026-06-09 10:03:20.566][16][debug][jwt] [source/extensions/filters/http/jwt_authn/filter.cc:150] Called Filter : setDecoderFilterCallbacks
[2026-06-09 10:03:20.566][16][debug][jwt] [source/extensions/filters/http/jwt_authn/filter.cc:54] Called Filter : decodeHeaders
[2026-06-09 10:03:20.566][16][debug][jwt] [source/extensions/filters/http/jwt_authn/filter.cc:109] Jwt authentication completed with: OK
[2026-06-09 10:03:20.566][16][debug][router] [source/common/router/router.cc:445] [C2][S5882164773424644491] cluster 'tc-trade-cluster' match for URL '/tcapi/abc'
[2026-06-09 10:03:20.566][16][debug][router] [source/common/router/router.cc:631] [C2][S5882164773424644491] router decoding headers:
':authority', 'localhost:7000'
':path', '/tcapi/abc'
':method', 'GET'
':scheme', 'http'
'user-agent', 'curl/8.7.1'
'accept', '*/*'
'x-forwarded-for', '192.168.215.1,192.168.215.1'
'x-forwarded-proto', 'http'
'x-envoy-internal', 'true'
'x-request-id', '930439bb-0403-47a2-98e0-c0f04345d21b'
'x-envoy-expected-rq-timeout-ms', '30000'

[2026-06-09 10:03:20.566][16][debug][pool] [source/common/http/conn_pool_base.cc:74] queueing stream due to no available connections
[2026-06-09 10:03:20.566][16][debug][pool] [source/common/conn_pool/conn_pool_base.cc:241] trying to create new connection
[2026-06-09 10:03:20.566][16][debug][pool] [source/common/conn_pool/conn_pool_base.cc:143] creating a new connection
[2026-06-09 10:03:20.566][16][debug][client] [source/common/http/codec_client.cc:43] [C3] connecting
[2026-06-09 10:03:20.566][16][debug][connection] [source/common/network/connection_impl.cc:860] [C3] connecting to 3.169.183.106:443
[2026-06-09 10:03:20.566][16][debug][connection] [source/common/network/connection_impl.cc:876] [C3] connection in progress
[2026-06-09 10:03:20.776][16][debug][connection] [source/common/network/connection_impl.cc:665] [C3] connected
[2026-06-09 10:03:20.948][16][debug][connection] [source/extensions/transport_sockets/tls/ssl_socket.cc:219] [C3] TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE 268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO
[2026-06-09 10:03:20.948][16][debug][connection] [source/common/network/connection_impl.cc:243] [C3] closing socket: 0
[2026-06-09 10:03:20.948][16][debug][connection] [source/extensions/transport_sockets/tls/ssl_socket.cc:219] [C3] TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE 268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO
[2026-06-09 10:03:20.948][16][debug][client] [source/common/http/codec_client.cc:101] [C3] disconnect. resetting 0 pending requests
[2026-06-09 10:03:20.948][16][debug][pool] [source/common/conn_pool/conn_pool_base.cc:393] [C3] client disconnected, failure reason: TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE 268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO
[2026-06-09 10:03:20.948][16][debug][router] [source/common/router/router.cc:1091] [C2][S5882164773424644491] upstream reset: reset reason: connection failure, transport failure reason: TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE 268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO
[2026-06-09 10:03:20.948][16][debug][http] [source/common/http/filter_manager.cc:883] [C2][S5882164773424644491] Sending local reply with details upstream_reset_before_response_started{connection failure,TLS error: 268436496:SSL routines:OPENSSL_internal:SSLV3_ALERT_HANDSHAKE_FAILURE 268435610:SSL routines:OPENSSL_internal:HANDSHAKE_FAILURE_ON_CLIENT_HELLO}
[2026-06-09 10:03:20.949][16][debug][http] [source/common/http/conn_manager_impl.cc:1455] [C2][S5882164773424644491] encoding headers via codec (end_stream=false):
':status', '503'
'access-control-allow-methods', '*'
'access-control-allow-origin', '*'
'access-control-allow-headers', '*'
'x-request-id', '930439bb-0403-47a2-98e0-c0f04345d21b'
'content-length', '303'
'content-type', 'application/json'
'date', 'Tue, 09 Jun 2026 10:03:20 GMT'
'server', 'envoy'

[2026-06-09 10:03:20.951][16][debug][jwt] [source/extensions/filters/http/jwt_authn/filter.cc:47] Called Filter : onDestroy
[2026-06-09 10:03:20.953][16][debug][connection] [source/common/network/connection_impl.cc:633] [C2] remote close
```

# SNI 是什么

**SNI = Server Name Indication（服务器名称指示）**，TLS 协议的一个扩展（RFC 6066），作用是：在 TLS 握手的第一步 `ClientHello` 里，**客户端明文告诉服务器"我要访问的域名是什么**，这样服务器才能挑出对应的证书返回。

## 为什么需要 SNI

回到 HTTPS 的本质矛盾：

- **HTTP** 的 `Host` 头是在 TCP 连接建立**之后**才发的，所以一个 IP 可以靠 `Host` 区分多个站点（虚拟主机）。
- **HTTPS** 必须**先**完成 TLS 握手才能跑 HTTP，而 TLS 握手时服务器要立刻把证书发给客户端。

问题来了：如果一个 IP（比如某台 nginx、某个 CDN 节点、AWS ALB）上挂了 100 个域名、100 张证书，**服务器在握手那一刻怎么知道该用哪张证书？**

SNI 就是解决这个的——客户端在 ClientHello 里塞一个字段：

```
server_name: api.tradingcentral.com
```

服务器一看，"哦你要访问 api.tradingcentral.com，给你这张证书"，然后握手继续。

如果客户端**不带 SNI**：

- 老式单域名服务器：用默认证书继续握手，可能能成；
- 现代 CDN / 云负载均衡（CloudFront / AWS ALB / Cloudflare / Akamai / 阿里云 CDN…）：**直接断开，返回 `handshake_failure` alert**。

## 一次握手中 SNI 长什么样

抓包看 ClientHello 的 Extensions 部分：

```
Extension: server_name (len=24)
    Server Name Indication extension
        Server Name list length: 22
        Server Name Type: host_name (0)
        Server Name length: 19
        Server Name: api.tradingcentral.com
```

注意：

- **SNI 是明文的**（TLS 1.3 之前都是，1.3 + ECH 才能加密）。所以中间盒、防火墙能基于 SNI 做策略，比如 GFW 阻断、企业出口审计、CDN 路由。
- **SNI 里只能写域名，不能写 IP**（RFC 要求），写 IP 的话很多 server 会忽略或报错。
- **SNI 不带端口、不带 scheme、不带路径**——就是个裸域名字符串。

## SNI vs Host vs 证书 CN/SAN：别搞混

这三者用在不同阶段，但通常应该一致：

| 名称 | 在哪里 | 谁发的 | 作用 |
|------|--------|--------|------|
| **SNI** | TLS ClientHello（握手阶段） | 客户端 | 让 server 选证书 |
| **Host / `:authority`** | HTTP 请求头（握手之后） | 客户端 | 让 server 选虚拟主机/路由 |
| **证书 CN / SAN** | 服务器证书里 | 服务器（CA 签发） | 客户端用来校验"这张证书是不是给这个域名的" |

握手的完整流程（简化）：

```
Client                                    Server
  | --- ClientHello (SNI=api.x.com) --->  |  ← server 用 SNI 挑证书
  | <-- ServerHello + Certificate ------  |
  |     (证书 SAN 里包含 api.x.com)        |  ← client 用 SAN 校验
  | --- (TLS 握手完成) ------------------ |
  | --- HTTP GET / Host: api.x.com -----> |  ← server 用 Host 路由
```

三个值正常情况下都是 `api.x.com`，**但允许不一样**：比如代理转发场景，SNI 可以是 `backend.internal`，Host 可以是 `api.public.com`。

## 在 Envoy 里 SNI 是怎么配的

### 1. 上游方向（envoy 当 client 去连接外部服务）

配在 `cluster.transport_socket`：

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    sni: api.tradingcentral.com     # ← 这个
    common_tls_context:
      validation_context:
        match_subject_alt_names:
          - exact: api.tradingcentral.com
        trusted_ca:
          filename: /etc/ssl/certs/ca-certificates.crt
```

也可以让 envoy 自动用客户端 `Host` 头当 SNI：

```yaml
upstream_http_protocol_options:
  auto_sni: true
  auto_san_validation: true
```

### 2. 下游方向（envoy 当 server，接受客户端 HTTPS 连接）

基于 SNI 给不同域名挂不同证书：

```yaml
filter_chains:
- filter_chain_match:
    server_names: ["api.gtja.com"]      # ← 按 SNI 匹配
  transport_socket: { ... 证书 A ... }
- filter_chain_match:
    server_names: ["trade.gtja.com"]
  transport_socket: { ... 证书 B ... }
```

## 用命令行验证 / 调试 SNI

```bash
# 带 SNI 握手，最常用
openssl s_client -connect api.tradingcentral.com:443 \
                 -servername api.tradingcentral.com </dev/null

# 不带 SNI（-servername "" 或不传），用来复现 handshake_failure
openssl s_client -connect api.tradingcentral.com:443 -servername "" </dev/null

# 用 curl 显式指定 SNI（curl 默认会用 URL 里的 host 当 SNI，
# 但当你用 --resolve 或直接连 IP 时就需要 --resolve 或 -H 配合）
curl -v --resolve api.tradingcentral.com:443:3.169.183.106 \
     https://api.tradingcentral.com/health
```

## 一句话总结

> **SNI 就是 TLS 握手时客户端预先告诉服务器"我要访问哪个域名"的那张名片，没这张名片，挂在同一个 IP 上的现代 server 就不知道该给你哪张证书，于是直接拒绝握手。**
