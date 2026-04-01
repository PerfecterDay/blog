#  Http gzip 压缩
{docsify-updated}

- [Http gzip 压缩](#http-gzip-压缩)
    - [Http压缩](#http压缩)
    - [压缩试验](#压缩试验)
    - [自己写的 Proxy 服务](#自己写的-proxy-服务)


### Http压缩

为了选择要采用的压缩算法，浏览器和服务器之间会使用主动协商机制。浏览器发送 `Accept-Encoding` 标头，其中包含有它所支持的压缩算法，以及各自的优先级，服务器则从中选择一种，使用该算法对响应的消息主体进行压缩，并且发送 `Content-Encoding` 标头来告知浏览器它选择了哪一种算法。由于该内容协商过程是基于编码类型来选择资源的展现形式的，在响应时，服务器至少发送一个包含 `Accept-Encoding` 的 Vary 标头以及该标头；这样的话，缓存服务器就可以对资源的不同展现形式进行缓存。

```
Accept-Encoding: gzip
Accept-Encoding: compress
Accept-Encoding: deflate
Accept-Encoding: br
Accept-Encoding: identity
Accept-Encoding: *

// Multiple algorithms, weighted with the quality value syntax:
Accept-Encoding: deflate, gzip;q=1.0, *;q=0.5
```
+ `gzip`: 表示采用 Lempel-Ziv coding (LZ77) 压缩算法，以及 32 位 CRC 校验的编码方式。
+ `compress`: 采用 Lempel-Ziv-Welch (LZW) 压缩算法。
+ `deflate`: 采用 zlib 结构和 deflate 压缩算法。
+ `br`: 表示采用 Brotli 算法的编码方式。
+ `identity`: 用于指代自身（例如：未经过压缩和修改）。除非特别指明，这个标记始终可以被接受。
+ `*`: 匹配其他任意未在该请求头字段中列出的编码方式。假如该请求头字段不存在的话，这个值是默认值。它并不代表任意算法都支持，而仅仅表示算法之间无优先次序。
+ `;q= (qvalues weighting)`:值代表优先顺序，用相对质量价值 表示，又称为权重。

### 压缩试验

`curl -vv https://www.baidu.com/ -H "Accept-Encoding: gzip,deflate"` : gzip 格式的响应
<center><img src="pics/http-gzip1.jpg" width="70%"></center>


`curl https://www.baidu.com/ -H "Accept-Encoding: gzip,deflate" | gunzip` 或者 `curl https://www.baidu.com/ -H "Accept-Encoding: gzip,deflate" --compressed`
<center><img src="pics/http-gzip2.jpg" width="70%"></center>


### 自己写的 Proxy 服务
```
 @PostMapping("/**")
public ResponseEntity<String> proxy(@RequestBody String body, HttpServletRequest servletRequest, HttpServletResponse response) {
    try {
//            HttpMethod method = HttpMethod.resolve(servletRequest.getMethod());
        HttpMethod method = HttpMethod.GET;
        String uriStr = servletRequest.getRequestURI().replace("/app/user", "");
        URI uri = new URI("https://www.baidu.com");

        HttpHeaders headers = new HttpHeaders();
        Enumeration<String> headerNames = servletRequest.getHeaderNames();

        while (headerNames.hasMoreElements()) {
            String headerName = headerNames.nextElement();
            headers.set(headerName, servletRequest.getHeader(headerName));
        }

        HttpEntity<String> httpEntity = new HttpEntity<>(body, headers);
        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> serverResponse = restTemplate.exchange(uri, method, httpEntity, String.class);
        return serverResponse;

    } catch (URISyntaxException e) {
        throw new RuntimeException(e);
    }
}
```

```
curl -vv --location 'http://localhost:8050/user/exclusiveGift/aa' \
--header 'Content-Type: application/json' \
--data '{
    "pageNum": 1,
    "pageSize": 10
}'
```
<center><img src="pics/http-gzip5.jpg" width="70%"></center>



```
curl -vv --location 'http://localhost:8050/user/exclusiveGift/aa' \
--header 'Content-Type: application/json' \
--header 'Accept-Encoding: gzip,deflate' \
--data '{
    "pageNum": 1,
    "pageSize": 10
}'
```
<center><img src="pics/http-gzip3.jpg" width="70%"></center>


```
curl -vv --location 'http://localhost:8050/user/exclusiveGift/aa' \
--header 'Content-Type: application/json' \
--header 'Accept-Encoding: gzip,deflate' \
--data '{
    "pageNum": 1,
    "pageSize": 10
}' --compressed
```
<center><img src="pics/http-gzip4.jpg" width="70%"></center>
