# Envoy 集成 JWT token 验证
{docsify-updated}

> https://mkjwk.org/  
> https://jwt.io/  
> https://github.com/jwtk/jjwt  

Envoy 使用 HTTP 过滤器来实现 JWT token 的验证。 它将验证其 signature 、 audiences 和 issuer 。 它还将检查其时间限制，如过期时间和 nbf（非之前）时间。 如果 JWT 验证失败，其请求将被拒绝。 如果 JWT 验证成功，其有效载荷可以转发给上游，以便根据需要进一步授权。

验证 JWT 签名需要 JWKS 。 它们可以在过滤器配置中指定，也可以从 JWKS 服务器远程获取。JSON WEB KEY（JWK）是一种 JavaScript Object Notation（JSON）数据结构规范，用于表示加密密钥。 该规范定义了表示一组 JWKs 的 JWK Set JSON 数据结构。  本规范使用的加密算法和标识符在单独的 JSON Web 算法（JWA）规范和该规范建立的 IANA 注册表中进行了说明。

## 配置

```yml
http_filters:
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
    providers:
      jwt_provider1:
        issuer: https://example.com
        audiences:
          audience1
        local_jwks:
          inline_string: "{\"keys\": [{\"kty\": \"EC\",\"d\": \"NjSAQILonTTNZLArIX2j6MAFzSGWIbypGptfeuGEnZU\",\"use\": \"sig\",\"crv\": \"P-256\",\"kid\": \"LJj6hpmMmTZvrTZm1kiTdbHa9vMATQBSe_rfe36EnB0\",\"x\": \"0NHrttPMVGa-yLOujJHVa5MxjsIqdYuorPEHmJxQzQ0\",\"y\": \"lCXoQh5ctrnyDFmuceGT_QGMqrC6CQGFd1zjIW32WAE\",\"alg\": \"ES256\"}]}"
        from_headers:
          name: jwt-assertion
        forward: true
        forward_payload_header: x-jwt-payload
    rules:
    - match:
        prefix: /health
    - match:
        prefix: /api
      requires:
        provider_and_audiences:
          provider_name: jwt_provider1
          audiences:
            api_audience
    - match:
        prefix: /
      requires:
        provider_name: jwt_provider1
```

JWT 验证过滤器配置由两部分组成：
1. `providers`: 规定了 JWT 的验证方式，例如在哪里提取token、在哪里获取公钥（JWKS）以及在哪里输出其有效载荷。
2. `rules`: 字段指定了匹配规则及其对应需要使用的 `provider` 。

### JwtProvider
JwtProvider 规定了如何验证 JWT 的方式。 它有以下字段：

+ `issuer`: （签发者）：签发 JWT 的委托人，通常是一个 URL 或电子邮件地址。 
+ `audiences`: （受众）：允许访问的 JWT 受众列表。 包含这些受众中任何一个的 JWT 都会被接受。 如果未指定，则不会检查 JWT 中的受众。 
+ `local_jwks`: 从本地数据源获取 JWKS，可以是本地文件，也可以是内嵌字符串。 
+ `remote_jwks`: 从远程 HTTP 服务器获取 JWKS，也可指定缓存持续时间。 
+ `forward`: 如果为 true，JWT 将被转发到上游。
+ `from_headers`: 从 HTTP 头信息中提取 JWT。 
+ `from_params`: 从查询参数中提取 JWT。 
+ `from_cookies`: 从 HTTP 请求 cookie 中提取 JWT。 
+ `forward_payload_header`: 将 JWT 有效载荷转发到指定的 HTTP 头信息中。 
+ `claim_to_headers`: 将 JWT 索赔复制到 HTTP 头信息中： 启用 JWT 缓存，其大小可由 
+ `jwt_cache_size`: 指定。 只有有效的 JWT 标记才会被缓存。

如果 `from_headers` 和 `from_params` 为空，提取 JWT 的默认位置就是 HTTP 头：
```
Authorization: Bearer <token>
```
和查询参数 key access_token 为：
```
/path?access_token=<JWT>
```
如果一个请求有两个token，一个来自标头，另一个来自查询参数，那么所有token都必须有效。 