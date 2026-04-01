# Okhttp简介
{docsify-updated}

> https://segmentfault.com/a/1190000045343278#item-3-3

- [Okhttp简介](#okhttp简介)
  - [核心类及作用](#核心类及作用)
  - [Get 请求](#get-请求)
  - [Post 请求](#post-请求)
  - [HttpUrl.Builder-构造带参数的 URL](#httpurlbuilder-构造带参数的-url)
  - [异步发送请求](#异步发送请求)
  - [设置超时时间](#设置超时时间)
  - [设置连接池](#设置连接池)


Okhttp是一个处理网络请求的开源项目,是安卓端最火热的轻量级框架,由移动支付Square公司贡献(该公司还贡献了Picasso)用于替代 HttpUrlConnection 和 Apache HttpClient (android API23 6.0里已移除HttpClient,现在已经打不出来)

在现代应用程序网络处理中，HTTP是我们交换数据和媒体的方式。高效地执行HTTP可以使您的数据加载更快并节省带宽。

在默认情况下，OKHTTP是高效的HTTP客户端：
1. HTTP/2支持允许对同一主机的所有请求共享一个套接字。
2. 连接池减少了请求延迟（如果HTTP/2不可用）。
3. 透明gzip缩小下载大小。
4. 响应缓存完全避免了重复请求的网络。

当网络出现问题时，OKHTTP会坚持下去：它会自动从常见的连接问题中恢复。如果您的服务有多个IP地址，在第一次连接失败时，OKHTTP将尝试备用地址。这对于IPv4+IPv6和托管在冗余数据中心中的服务是必需的。OKHTTP支持现代TLS功能（TLS 1.3、ALPN、证书固定）。它可以配置为回调以实现广泛的连接。

使用OKHTTP很容易。它的request/response API设计为具有流畅的构建者模式和不变性。它支持同步阻塞调用和带回调的异步调用。

OKHTTP支持Android 5 +（API级别21 +）和Java 8 +。

## 核心类及作用
+ `Request` : 代表一个 Http 请求实体信息，可以多次发送一个 Request，每次 Request 相同，但是 Call 不同
+ `Request.Builder` : 用来创建 Request 的 builder
  + `addHeader("Content-Type", "application/json")`: 添加请求头
+ `HttpUrl` : 代表 Http URL
+ `HttpUrl.Builder` : 用来创建 HttpUrl 的 builder
  + `addQueryParameter(String paramName, String paramValue)`
  + `HttpUrl.new Builder()`: 创建 builder
+ `RequestBody` : 代表一个请求体内容，通常用于 POST 请求，常用的有 `FormBody` 和 `MultipartBody`
  + `new FormBoday.Builder().build()`
  + `new MultipartBody.Builder().build()`
  + `RequestBody.create(json, JSON)`
+ `OkHttpClient` : 核心类，用来发送Http 请求
+ `Call` : 代表每一次 Http 请求调用过程，可以多次发送一个 Request，每次 Request 相同，但是 Call 不同 
  + `client.newCall(Reqeust request)` ： 生成一个 Call 对象
  + `execute()`： 同步发送 http 请求
  + `enqueue(Callback callback)` ： 异步发送 Http 请求
+ `OkHttpClient.Builder` : 创建 OkHttpClient 的 builder

## Get 请求
发起一个 Get 请求时简单的：
```
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();
  
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## Post 请求
```
public static final MediaType JSON
    = MediaType.get("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(json, JSON);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## HttpUrl.Builder-构造带参数的 URL
```
HttpUrl.Builder urlBuilder 
  = HttpUrl.parse(BASE_URL + "/ex/bars").newBuilder();
urlBuilder.addQueryParameter("id", "1");

String url = urlBuilder.build().toString();
```

## 异步发送请求
```
Request request = new Request.Builder()
    .url(BASE_URL + "/date")
    .build();

Call call = client.newCall(request);
call.enqueue(new Callback() {
    public void onResponse(Call call, Response response) 
      throws IOException {
        // ...
    }
    
    public void onFailure(Call call, IOException e) {
        fail();
    }
});
```

## 设置超时时间
```
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS) //连接超时时间
    .writeTimeout(10, TimeUnit.SECONDS) // 写超时时间
    .readTimeout(30, TimeUnit.SECONDS) // 读超时时间
    .build();

Request request = new Request.Builder()
  .url(BASE_URL + "/delay/2")
  .build();

Call call = client.newCall(request);
Response response = call.execute();
```

## 设置连接池
```
return new okhttp3.OkHttpClient.Builder().connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES)) // 最大50个空闲连接，保活5分钟
    .connectTimeout(10, TimeUnit.SECONDS).readTimeout(20, TimeUnit.SECONDS).writeTimeout(20, TimeUnit.SECONDS)
    .build();
```