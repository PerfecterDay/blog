#  浏览器的同源策略与跨源资源共享（CORS）
{docsify-updated}

> https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS  
> https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy

## 同源策略
同源策略是一个重要的安全策略，它用于限制一个源的文档或者它加载的脚本如何能与另一个源的资源进行交互。

它能帮助阻隔恶意文档，减少可能被攻击的媒介。例如，它可以防止互联网上的恶意网站在浏览器中运行 JS 脚本，从第三方网络邮件服务（用户已登录）或公司内网（因没有公共 IP 地址而受到保护，不会被攻击者直接访问）读取数据，并将这些数据转发给攻击者。

另一个需要注意的是拦截的时机。
+ 对于简单跨域请求，因为只有一次（没有 OPTIONS 方法的预检请求），所以请求会发到后端，如果触发同源限制，浏览器会拦截响应，引发 CORS 警告。
+ 对于非简单请求，如果预检 OPTIONS 请求的响应触发了同源限制，浏览器会拦截真实的请求，引发 CORS 警告。

所以两种情况下，后端都会收到请求，浏览器收到相应后，进行同源策略匹配，决定是否放行。

### 同源的的定义
如果两个 URL 的**协议、端口和主机**都相同的话，则这两个 URL 是同源的。

## CORS
**跨源资源共享（CORS，或通俗地译为跨域资源共享）是一种基于 HTTP 头的机制，该机制通过允许服务器标示除了它自己以外的其他源（域、协议或端口），使得浏览器允许这些源访问加载服务器的资源**。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的“预检”请求。在预检中，浏览器发送的头中标示有 HTTP 方法和真实请求中会用到的头。

跨源 HTTP 请求的一个例子：运行在 https://domain-a.com 的 JavaScript 代码使用 XMLHttpRequest 来发起一个到 https://domain-b.com/data.json 的请求。

出于安全性，浏览器限制脚本内发起的跨源 HTTP 请求。例如，XMLHttpRequest 和 Fetch API 遵循同源策略。这意味着使用这些 API 的 Web 应用程序只能从加载应用程序的同一个域请求 HTTP 资源，除非响应报文包含了正确 CORS 响应头。

**跨源资源共享标准新增了一组 HTTP 标头字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。另外，规范要求，对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 OPTIONS 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨源请求(chrome 网络全部或其它里边能看到类型为preflight的请求)。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的响应中，服务器端也可以通知客户端，是否需要携带身份凭证（例如 Cookie 和 HTTP 认证相关数据）**。

涉及CORS的请求会被分成几类：
+ 简单请求（不会产生预检请求）
+ 预检请求
+ 真实请求
+ 包含 credentials 的请求

### 简单请求
某些请求不会触发 CORS 预检请求。在废弃的 CORS 规范中称这样的请求为简单请求。  
若请求满足所有下述条件，则该请求可视为简单请求：
1. 使用下列方法之一：`GET/HEAD/POST`
2. 除了被用户代理自动设置的标头字段（例如 Connection、User-Agent 或其他在 Fetch 规范中定义为禁用标头名称的标头），只设置了 Fetch 规范定义的对 CORS 安全的标头集合：`Accept、 Accept-Language、 Content-Language、 Content-Type(只能是 text/plain  multipart/form-data application/x-www-form-urlencoded 之一)、 Range` 
3. 如果请求是使用 XMLHttpRequest 对象发出的，在返回的 XMLHttpRequest.upload 对象属性上没有注册任何事件监听器；也就是说，给定一个 XMLHttpRequest 实例 xhr，没有调用 xhr.upload.addEventListener()，以监听该上传请求。
4. 请求中没有使用 ReadableStream 对象。

对于简单请求，服务器仍然必须提供 `Access-Control-Allow-Origin`等的选择，仍然受同源策略的限制，只不过不会产生预检请求。以便与脚本共享响应。

### 预检请求
与简单请求不同，“需预检的请求”要求必须首先使用 `OPTIONS` 方法发起一个预检请求到服务器，以获知服务器是否允许该实际请求。"预检请求“的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。
preflight 请求不会发送自定义的请求头，因为预检正是要检测这些请求头是否符合要求，如果发送了就失去预检的意义了。所以，如果依赖请求头的场景需要注意。比如：

envoy 配置了 jwt 校验，取的请求头中的 jwt token。这时后如果跨域发起预检请求，不会发送这个请求头，envoy 会返回401，导致预检失败。所以要在 envoy jwt 校验那块过滤掉 OPTIONS 方法。

要启用CORS，预检请求的响应和实际请求的响应都需要包含相关的 `Access-Control-Allow-*` 头部，并且这两个头部的值应该是相同的。

### 附带身份凭证的请求

### CORS 相关的Http头
XMLHttpRequest 或 Fetch 与 CORS 的一个有趣的特性是，可以基于 HTTP cookies 和 HTTP 认证信息发送身份凭证。一般而言，对于跨源 XMLHttpRequest 或 Fetch 请求，浏览器不会发送身份凭证信息。如果要发送凭证信息，需要设置 XMLHttpRequest 对象的某个特殊标志位，或在构造 Request 对象时设置。
```
https://appapi.demo.com/yichat/broker/info?token=1230452636424077312_dba96b34f42145a992f767dd0fedcd88&t=1713404369679

var xhr = new XMLHttpRequest();
xhr.open("GET", "https://uat-mobileapp.demo.com/yichat/broker/info?token=1230466734977187840_24499053f2f5405598c7949da67fc76e&t=1", true);
xhr.send(null);

var xhr = new XMLHttpRequest();
xhr.open("GET", "https://uat-mobileapp.demo.com/yichat/broker/info?token=1230466734977187840_24499053f2f5405598c7949da67fc76e&t=1", true);
xhr.withCredentials = true;
xhr.send(null);

fetch(url, {
  credentials: "include",
});
```

#### 响应头
##### Access-Control-Allow-Origin
Access-Control-Allow-Origin 响应标头指定了该响应的资源是否被允许与给定的来源（origin）共享。
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Origin: <origin>
Access-Control-Allow-Origin: null
```
+ `*`
    对于不包含凭据的请求，服务器会以“*”作为通配符，从而允许任意来源的请求代码都具有访问资源的权限。尝试使用通配符来响应包含凭据的请求会导致错误。
+ `<origin>`
    指定一个来源（只能指定一个）。如果服务器支持多个来源的客户端，其必须以与指定客户端匹配的来源来响应请求。
+ `null`

如果服务端指定了具体的单个源而非通配符“*”（作为允许列表的一部分，服务器可能会根据请求的来源而动态改变），那么响应标头中的 Vary 字段的值必须包含 Origin。这将告诉客户端：服务器对不同的 Origin 返回不同的内容。
```
Access-Control-Allow-Origin: https://mozilla.org
Vary: Origin
```

##### Access-Control-Allow-Methods
Access-Control-Allow-Methods 表明服务器允许客户端使用的请求方法。  
`Access-Control-Allow-Methods: POST, GET, OPTIONS` 表明服务器允许客户端使用 POST 和 GET 方法发起请求

##### Access-Control-Allow-Headers
Access-Control-Allow-Headers 表明服务器允许请求中携带的请求头。  
`Access-Control-Allow-Headers: X-PINGOTHER, Content-Type` 表明服务器允许请求中携带字段 X-PINGOTHER 与 Content-Type 。

#####  Access-Control-Max-Age
Access-Control-Max-Age 给定了该预检请求可供缓存的时间长短，单位为秒，默认值是 5 秒。在有效时间内，浏览器无须为同一请求再次发起预检请求。请注意，浏览器自身维护了一个最大有效时间，如果该标头字段的值超过了最大有效时间，将不会生效。

##### Access-Control-Allow-Credentials
Access-Control-Allow-Credentials 响应头用于在请求要求包含 credentials 时（附带身份凭证的请求），告知浏览器是否可以将响应暴露给前端 JavaScript 代码。  
CORS 预检请求不能包含凭据。预检请求的响应必须指定 Access-Control-Allow-Credentials: true 来表明可以携带凭据进行实际的请求。
请注意：简单 GET 请求不会被预检；如果对此类请求的响应中不包含该字段，这个响应将被忽略掉，并且浏览器也不会将相应内容返回给网页。如果服务器端的响应中未携带 `Access-Control-Allow-Credentials: true`，浏览器将不会把响应内容返回请求者。

另外，在响应附带身份凭证的请求时：
+ 服务器不能将 `Access-Control-Allow-Origin` 的值设为通配符“*”，而应将其设置为特定的域，如：Access-Control-Allow-Origin: https://example.com。
+ 服务器不能将 `Access-Control-Allow-Headers` 的值设为通配符“*”，而应将其设置为标头名称的列表，如：Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
+ 服务器不能将 `Access-Control-Allow-Methods` 的值设为通配符“*”，而应将其设置为特定请求方法名称的列表，如：Access-Control-Allow-Methods: POST, GET

对于附带身份凭证的请求（通常是 Cookie），这是因为请求的标头中携带了 Cookie 信息，如果 Access-Control-Allow-Origin 的值为“*”，请求将会失败。而将 Access-Control-Allow-Origin 的值设置为 https://example.com，则请求将成功执行。

另外，响应标头中也携带了 Set-Cookie 字段，尝试对 Cookie 进行修改。如果操作失败，将会抛出异常。

##### Access-Control-Expose-Headers
在跨源访问时，XMLHttpRequest 对象的 getResponseHeader() 方法只能拿到一些最基本的响应头: `Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma`，如果要访问其他头，则需要服务器设置本响应头。

Access-Control-Expose-Headers 头将指定标头放入允许列表中，供浏览器的 JavaScript 代码（如 getResponseHeader()）获取。
```
Access-Control-Expose-Headers: X-My-Custom-Header, X-Another-Custom-Header
```
这样浏览器就能够通过 getResponseHeader 访问 X-My-Custom-Header 和 X-Another-Custom-Header 响应头了。

#### 请求头
可用于发起跨源请求的标头字段。请注意，这些标头字段无须手动设置。当开发者使用 XMLHttpRequest 对象发起跨源请求时，它们已经被设置就绪。

##### Origin 
Origin 标头字段表明预检请求或实际跨源请求的源站。它不包含任何路径信息，只是服务器名称。在所有访问控制请求中，Origin 标头字段总是被发送。

##### Access-Control-Request-Method
Access-Control-Request-Method 标头字段用于预检请求。其作用是，将实际请求所使用的 HTTP 方法告诉服务器。

##### Access-Control-Request-Headers
Access-Control-Request-Headers 标头字段用于预检请求。其作用是，将实际请求所携带的标头字段（通过 setRequestHeader() 等设置的）告诉服务器。这个浏览器端标头将由互补的服务器端标头 Access-Control-Allow-Headers 回答。