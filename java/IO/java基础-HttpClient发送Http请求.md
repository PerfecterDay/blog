# 使用Http Client API发送 Http 请求
{docsify-updated}

> https://www.baeldung.com/java-9-http-client  
> https://openjdk.org/groups/net/httpclient/intro.html

`HttpClient` 是在 Java 11 中新增的。它可用于通过网络请求 HTTP 资源。它支持 HTTP/1.1、HTTP/2 和 HTTP/3，同时支持同步和异步编程模型，将请求和响应正文作为响应流进行处理，并遵循大家熟悉的构建器模式。

```
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("http://openjdk.org/"))
      .build();
client.sendAsync(request, BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .join();
```


## HttpClient
要发送请求，首先需要通过其构建器创建一个 `HttpClient` 。该构建器可用于配置每个客户端的状态，例如：
+ 首选的协议版本（HTTP/1.1、HTTP/2 或 HTTP/3）
+ 是否跟随重定向
+ 代理
+ 身份验证器
+ 连接超时时间
+ SSLContext
+ CookieHandler
+ 执行器

```
HttpClient client = HttpClient.newBuilder()
      .version(Version.HTTP_2)
      .connectTimeout(Duration.ofSeconds(1))
      .cookieHandler(CookieHandler.getDefault())
      .followRedirects(Redirect.NORMAL)
      .proxy(ProxySelector.of(new InetSocketAddress("www-proxy.com", 8080)))
      .authenticator(Authenticator.getDefault())
      .sslContext(SSLContext.getDefault())
      .executor(Executors.newFixedThreadPool(10))
      .build();
```

一旦创建了 `HttpClient` ，就可以使用它发送多个 Http 请求。

## HttpRequest
HttpRequest 通过其构建器创建。请求构建器可用于设置：
+ 请求 URI
+ 请求方法（GET、PUT、POST）
+ 请求正文（如有）
+ 超时时间
+ 请求头

```
HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("http://openjdk.org/"))
      .timeout(Duration.ofMinutes(1))
      .header("Content-Type", "application/json")
      .POST(BodyPublishers.ofFile(Paths.get("file.json")))
      .build();
``` 

`HttpRequest` 是不可以修改的，一旦创建完就可以被发送，可以发送多次。

## Synchronous or Asynchronous

Synchronous :
```
HttpResponse<String> response =
      client.send(request, BodyHandlers.ofString());
System.out.println(response.statusCode());
System.out.println(response.body());
```

异步 API 会立即返回一个 `CompletableFuture` ，当 `HttpResponse` 可用时，该 `CompletableFuture` `便会完成。CompletableFuture` 是 Java 8 中新增的，支持可组合的异步编程。
```
client.sendAsync(request, BodyHandlers.ofString())
      .thenApply(response -> { System.out.println(response.statusCode());
                               return response; } )
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);
```

## HTTP/2
Java HTTP 客户端支持 HTTP/1.1、HTTP/2 和 HTTP/3（自 JDK 26 起）。默认情况下，客户端将使用 HTTP/2 发送请求。发送到尚未支持 HTTP/2 的服务器的请求将自动降级为 HTTP/1.1。以下是 HTTP/2 带来的主要改进总结：
+ 标头压缩。HTTP/2 使用 HPACK 压缩，可减少开销。
+ 与服务器建立单一连接，减少了建立多个 TCP 连接所需的往返次数。
+ 多路复用。允许在同一连接上同时处理多个请求。
+ 服务器推送。可将客户端未来可能需要的额外资源提前发送给客户端。
+ 二进制格式。更紧凑。

由于 HTTP/2 是默认的首选协议，且实现会在必要时无缝回退到 HTTP/1.1，因此当 HTTP/2 得到更广泛部署时，Java HTTP 客户端将能够从容应对未来的需求。

## HTTP3
> https://inside.java/2025/10/22/http3-support/

```
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_3)
        .build();

HttpRequest req = HttpRequest.newBuilder()
        .uri(reqURI)
        .version(HttpClient.Version.HTTP_3)
        .build();
```

无论哪种情况，当将 HTTP_3 设置为首选版本时，HttpClient 实现都会尝试通过 UDP 与目标服务器建立连接（因为 HTTP/3 通过 UDP 运行）。如果基于 UDP 的 QUIC 连接尝试失败——无论是因为服务器不支持 HTTP/3，还是因为连接未能及时建立——HttpClient 实现将自动将协议版本降级为 HTTP/2（基于 TCP），并尝试使用 HTTP/2 完成请求。如果服务器不支持 HTTP/2，则请求将像以前一样进一步降级为 HTTP/1.1。

