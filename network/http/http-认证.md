# Http 认证
{docsify-updated}

##  通用的 HTTP 认证框架
RFC 7235 定义了一个 HTTP 身份验证框架，服务器可以用来质询（challenge）客户端的请求，客户端则可以提供身份验证凭据。

质询与响应的工作流程如下：

+ 服务器端向客户端返回 401（Unauthorized，未被授权的）响应状态码，并在 `WWW-Authenticate` 响应标头提供如何进行验证的信息，其中至少包含有一种质询方式。
+ 之后，想要使用服务器对自己身份进行验证的客户端，可以通过包含凭据的 `Authorization` 请求标头进行验证。
+ 通常，客户端浏览器会向用户显示密码提示，然后发送包含正确的 `Authorization` 标头的请求。

上述整体的信息流程，对于大多数（并非是全部）身份验证方案都是相同的。标头中的真实信息和编码的方式确实发生了变化。

### 代理认证
与上述同样的询问质疑和响应原理适用于代理认证。由于资源认证和代理认证可以并存，区别于独立的标头和响应状态码。对于代理，询问质疑的状态码是 407（必须提供代理证书），响应标头 `Proxy-Authenticate` 至少包含一个可用的质询，并且请求标头 `Proxy-Authorization` 用作向代理服务器提供凭据。


```
WWW-Authenticate: <type> realm=<realm>
Authorization: <type> <credentials>

Proxy-Authenticate: <type> realm=<realm>
Proxy-Authorization: <type> <credentials>
```

### 禁止访问
如果（代理）服务器收到无效的凭据，它应该响应 `401 Unauthorized` 或 `407 Proxy Authentication Required` ，用户可以发送新的请求或替换 `Authorization` 标头字段。

如果（代理）服务器接受的有效凭据不足以访问给定的资源，服务器将响应 `403 Forbidden `状态码。与 `401 Unauthorized` 或 `407 Proxy Authentication Required` 不同的是，该用户无法进行身份验证并且浏览器不会提出新的尝试。

在所有情况下，服务器更可能返回 `404 Not Found` 状态码，以向没有足够权限或者未正确身份验证的用户隐藏页面的存在。

## 身份验证方案
通用 HTTP 身份验证框架可以被多个验证方案使用。不同的验证方案会在安全强度以及在客户端或服务器端软件中可获得的难易程度上有所不同。常见的验证方案包括：
+ Basic
+ Bearer
+ Digest
+ HOBA
+ Mutual
+ Negotiate / NTLM
+ VAPID
+ SCRAM
+ AWS4-HMAC-SHA256
