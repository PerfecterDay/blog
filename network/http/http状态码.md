# Http 状态码
{docsify-updated}

<center><img src="pics/http-status-code.png" width="50%"></center>

## 1XX
+ 100 Continue
+ 101 Switching Protocol
+ 102 Processing
+ 103 Early Hints

## 2XX
+ 200 OK
+ 201 Created
+ 202 Accepted
+ 203 Non-Authoritative Information
+ 204 No Content
+ 205 Reset Content
+ 206 Partial Content
+ 207 Multi-Status
+ 208 Already Reported
+ 226 IM Used

## 3XX
+ 300 Multiple Choices
+ 301 Moved Permanently
+ 302 Found
+ 303 See Other
+ 304 Not Modified
+ 307 Temporary Redirect
+ 308 Permanent Redirect

## 4XX
+ 400 Bad Request
+ 401 Unauthorized
+ 402 Payment Required
+ 403 Forbidden
+ 404 Not Found
+ 405 Method Not Allowed
+ 406 Not Acceptable
+ 407 Proxy Authentication Required
+ 408 Request Timeout
+ 409 Conflict
+ 410 Gone
+ 411 Length Required
+ 412 Precondition Failed
+ 413 Content Too Large
+ 414 URI Too Long
+ 415 Unsupported Media Type
+ 416 Range Not Satisfiable
+ 417 Expectation Failed
+ 418 I'm a teapot
+ 421 Misdirected Request
+ 422 Unprocessable Entity
+ 423 Locked
+ 424 Failed Dependency
+ 425 Too Early
+ 426 Upgrade Required
+ 428 Precondition Required
+ 429 Too Many Requests
+ 431 Request Header Fields Too Large
+ 451 Unavailable For Legal Reasons

## 5XX
+ 500 Internal Server Error ： 500（内部服务器错误）状态码表示服务器遇到意外情况，导致无法完成请求。
+ 501 Not Implemented ： 501（未实现）状态码表示服务器不支持满足请求所需的功能。当服务器无法识别请求方法且无法为任何资源提供支持时，此响应最为恰当。
+ 502 Bad Gateway : 502（网关错误）状态码表示服务器在充当中继或代理时，为处理请求而访问的上游服务器返回了无效响应。
+ 503 Service Unavailable
+ 504 Gateway Timeout ： 504（网关超时）状态码表示服务器在充当中继或代理时，未能及时收到上游服务器的响应，而该响应是完成请求所必需的。
+ 505 HTTP Version Not Supported ： HTTP 版本不适用
+ 506 Variant Also Negotiates
+ 507 Insufficient Storage : 存储空间不足
+ 508 Loop Detected
+ 510 Not Extended
+ 511 Network Authentication Required
+ 522 Connection Timed Out: The HTTP response status code 522 is an unofficial HTTP status code specific to Cloudflare. The 522 error occurs when Cloudflare times out contacting the origin web server