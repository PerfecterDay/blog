# Springboot 优雅日志打印库-logbook
{docsify-updated}

> https://github.com/zalando/logbook

## 核心组件
```
Logbook logbook = Logbook.builder()
    .condition(new CustomCondition())
    .attributeExtractor(new OriginExtractor())
    .queryFilter(new CustomQueryFilter())
    .pathFilter(new CustomPathFilter())
    .headerFilter(new CustomHeaderFilter())
    .bodyFilter(new CustomBodyFilter())
    .requestFilter(new CustomRequestFilter())
    .responseFilter(new CustomResponseFilter())
    .sink(new DefaultSink(
            new CustomHttpLogFormatter(),
            new CustomHttpLogWriter()
    ))
    .build();
```

### Strategy
Logbook 过去在请求/响应日志记录方面曾采用一种非常严格的策略：
+ 请求/响应分别进行记录
+ 请求/响应会尽快进行记录
+ 请求/响应要么成对记录，要么完全不记录（即不进行流量的局部记录）

虽然可以通过自定义的 `HttpLogWriter` 实现来缓解其中一些限制，但这些方案从来都不是理想的。

从 2.0 版本开始， `Logbook` 的核心采用了策略模式。 `Logbook` 内置了一些策略：
+ `BodyOnlyIfStatusAtLeastStrategy`
+ `StatusAtLeastStrategy`
+ `WithoutBodyStrategy`

### Attribute Extractor
从 3.4.0 版本开始， `Logbook` 新增了一项名为“属性提取器”（Attribute Extractor）的功能。属性本质上是一组键值对，可以从请求和/或响应中提取，并随其一起记录。这一想法源于问题 381，该问题中有人请求添加一项功能，用于从授权头中的 JWT 令牌中提取主体声明。

`AttributeExtractor` 接口提供了两个提取方法：一个仅能从请求中提取属性，另一个则同时支持请求和响应。这两个方法都返回 `HttpAttributes` 类的实例，该类本质上是一个高级的 `Map<String, Object>` 。请注意，由于映射的值类型为 `Object` ，因此它们应具备合适的 `toString()` 方法，才能在日志中以有意义的方式显示。此外，日志格式化器也可以通过实现自己的序列化逻辑来解决这个问题。例如，内置的日志格式化器 `JsonHttpLogFormatter` 便使用 `ObjectMapper` 来序列化这些值。

```
final class OriginExtractor implements AttributeExtractor {

  @Override
  public HttpAttributes extract(final HttpRequest request) {
    return HttpAttributes.of("origin", request.getOrigin());
  }
    
}
```

`JwtFirstMatchingClaimExtractor`/`JwtAllMatchingClaimsExtractor`. 前者从请求的 JWT 令牌中提取第一个与声明名称列表匹配的声明；后者则从请求的 JWT 令牌中提取所有与声明名称列表匹配的声明。

如果您需要组合多个 `AttributeExtractor` ，可以使用 `CompositeAttributeExtractor` 类：
```
final List<AttributeExtractor> extractors = List.of(
    extractor1,
    extractor2,
    extractor3
);

final Logbook logbook = Logbook.builder()
        .attributeExtractor(new CompositeAttributeExtractor(extractors))
        .build();
```

### Phases
`Logbook` 的执行分为几个不同的阶段：
1. `Conditional`
2. `Filtering`
3. `Formatting`
4. `Writing`

每个阶段由一个或多个接口表示，实现这些接口可用于自定义逻辑。每个阶段都有一个合理的默认值。

#### Conditional
记录 HTTP 消息并包含其正文是一项相当耗费资源的操作，因此针对某些请求禁用日志记录是非常合理的。一个常见的用例是忽略来自负载均衡器的健康检查请求，或是通常由开发人员发起的针对管理端点的任何请求。

定义一个条件非常简单，只需编写一个特殊的 `Predicate` ，用于判断请求（及其对应的响应）是否应被记录。此外，还可以使用并组合预定义的 `Predicate`：
```
Logbook logbook = Logbook.builder()
    .condition(exclude(
        requestTo("/health"),
        requestTo("/admin/**"),
        contentType("application/octet-stream"),
        header("X-Secret", newHashSet("1", "true")::contains)))
    .build();
```

#### Filtering
`Filtering` 的目的是防止记录HTTP请求和响应中的某些敏感内容。这通常包括 `Authorization` 标头，但也可能适用于某些明文查询参数或表单参数——例如密码。

| Type             | Operates on                    | Applies to | Default                                                                                            |
|------------------|--------------------------------|------------|----------------------------------------------------------------------------------------------------|
| `QueryFilter`    | Query string                   | request    | `access_token`                                                                                     |
| `PathFilter`     | Path                           | request    | n/a                                                                                                |
| `HeaderFilter`   | Header (single key-value pair) | both       | `Authorization`                                                                                    |
| `BodyFilter`     | Content-Type and body          | both       | json: `access_token` and `refresh_token`<br> form: `client_secret`, `password` and `refresh_token` |
| `RequestFilter`  | `HttpRequest`                  | request    | Replace binary, multipart and stream bodies.                                                       |
| `ResponseFilter` | `HttpResponse`                 | response   | Replace binary, multipart and stream bodies.                                                       |

`QueryFilter` 、 `PathFilter` 、 `HeaderFilter` 和 `BodyFilter` 属于相对高级的过滤器，在约 90% 的情况下应能满足所有需求。对于更复杂的配置，应改用低级变体，即分别使用 `RequestFilter` 和 `ResponseFilter` （配合 `ForwardingHttpRequest`/`ForwardingHttpResponse` 使用）。

```
Logbook logbook = Logbook.builder()
        .requestFilter(RequestFilters.replaceBody(message -> contentType("audio/*").test(message) ? "mmh mmh mmh mmh" : null))
        .responseFilter(ResponseFilters.replaceBody(message -> contentType("*/*-stream").test(message) ? "It just keeps going and going..." : null))
        .queryFilter(accessToken())
        .queryFilter(replaceQuery("password", "<secret>"))
        .headerFilter(authorization())
        .headerFilter(eachHeader("X-Secret"::equalsIgnoreCase, "<secret>"))
        .build();
```

#### Correlation
Logbook 使用 `correlation id ` 来关联请求和响应。这使得通常位于日志文件不同位置的关联请求和响应能够被匹配起来。 如果相关性 ID 的默认实现无法满足您的用例需求，可以提供自定义实现：
```
Logbook logbook = Logbook.builder()
    .correlationId(new CustomCorrelationId())
    .build();
```

#### Formatting
格式化功能基本上决定了请求和响应将如何转换为字符串。格式化器并不指定请求和响应的记录位置——这项工作由写入器负责。 日志记录器默认提供两种格式化器：HTTP 和 JSON。

#### Writing
`Writing` 用于指定格式化请求和响应的写入位置。Logbook 提供了三种实现方式： `Logger` 、 `Stream` 和 `Chunking` 。


#### Sink
`HttpLogFormatter` 与 `HttpLogWriter` 的组合能很好地满足大多数用例的需求，但它也有其局限性。直接实现 `Sink` 接口可以支持更复杂的用例，例如将请求/响应写入数据库等结构化持久化存储中。

使用 `CompositeSink` 可以将多个 `Sink` 合并为一个。

## 敏感字段脱敏指南
Logbook 4.0.4 提供两个层面的脱敏能力：**配置式**（够用 90% 场景）和**编程式**（处理复杂规则）。

### 一、配置式（推荐先用）

`LogbookProperties$Obfuscate` 支持 5 个字段：

| 字段 | 类型 | 作用 |
|---|---|---|
| `headers` | `List<String>` | 按名称匹配 Header（如 Authorization） |
| `parameters` | `List<String>` | 按名称匹配 Query 参数 |
| `paths` | `List<String>` | URL Path 段脱敏（按位置匹配） |
| `jsonBodyFields` | `List<String>` | **JSON Body 字段名脱敏** |
| `replacement` | `String` | 替换占位符（默认 `XXX`） |

完整示例：

```yaml
logbook:
  obfuscate:
    headers:
      - Authorization
      - Cookie
      - Set-Cookie
      - short-token
      - tmp-token
    parameters:
      - password
      - token
    paths:
      - /users/*/secret
    jsonBodyFields:
      - password
      - oldPassword
      - newPassword
      - idNumber
      - idCard
      - mobile
      - phone
      - bankAccount
      - creditCard
      - cvv
      - smsCode
      - otp
    replacement: "***"
```

效果示例：

```json
// 原始
{"name":"张三","mobile":"13800138000","password":"abc123"}
// 日志中
{"name":"张三","mobile":"***","password":"***"}
```

> 注意：`jsonBodyFields` 走的是 `PrimitiveJsonPropertyBodyFilter`，**只对 JSON 的基本类型字段（字符串/数字/布尔）生效**，对嵌套对象/数组不生效（要遮整个对象需用 JsonPath，见下文）。

### 二、编程式（处理复杂规则）

声明 `@Bean BodyFilter` / `@Bean HeaderFilter` 等，autoconfig 会自动把你的 Bean 与 yml 配置 **合并**（不会覆盖）。

#### 1. JsonPath 删除/替换（嵌套字段、数组）

```java
import static org.zalando.logbook.json.JsonPathBodyFilters.jsonPath;

@Configuration
class LogbookMaskConfig {
    @Bean
    BodyFilter jsonPathFilter() {
        return BodyFilter.merge(
            jsonPath("$.user.password").delete(),
            jsonPath("$..idCard").replace("***"),
            jsonPath("$.cards[*].cvv").replace("***"),
            jsonPath("$.mobile").replace("(\\d{3})\\d{4}(\\d{4})", "$1****$2")
        );
    }
}
```

#### 2. 按名称谓词替换（不写死字段名）

```java
@Bean
BodyFilter sensitiveJsonFilter() {
    Set<String> fields = Set.of("password", "token", "secret");
    return JsonBodyFilters.replaceJsonStringProperty(fields, "***");
}
```

#### 3. Header 高级规则

```java
@Bean
HeaderFilter cookieMask() {
    return HeaderFilters.replaceCookies(
        name -> name.startsWith("SESSION") || name.equals("JSESSIONID"),
        "***"
    );
}

@Bean
HeaderFilter authMask() {
    return HeaderFilters.authorization();
}
```

#### 4. OAuth access_token 一键脱敏

```java
@Bean BodyFilter oauthBody()  { return JsonBodyFilters.accessToken(); }
@Bean QueryFilter oauthQuery(){ return QueryFilters.accessToken(); }
```

#### 5. Form-urlencoded body

```java
@Bean
BodyFilter formMask() {
    return BodyFilters.replaceFormUrlEncodedProperty(
        Set.of("password", "token"), "***");
}
```

#### 6. 截断超大 body（防日志爆炸）

```java
@Bean BodyFilter truncate() { return BodyFilters.truncate(2000); }
```

### 三、合并语义提示

- `BodyFilter` / `HeaderFilter` / `QueryFilter` 类型的 Bean 可以定义**多个**，autoconfig 会用 `BodyFilter.merge(...)` 串起来 —— 你的 Bean 与 yml 配置共存，不互斥。
- 默认替换值 `XXX` 可在 yml 用 `logbook.obfuscate.replacement` 全局改；编程式则在每个 filter 里单独传。
- 顺序：filter 串接顺序对结果可能有影响（例如先 `compactXml` 再脱敏，与反过来不同），合并时注意排序。

### 四、部分脱敏（保留前后位）

纯 yml 做不到，必须走编程式：

```java
@Bean
BodyFilter mobilePartialMask() {
    return JsonBodyFilters.replacePrimitiveJsonProperty(
        name -> name.equals("mobile") || name.equals("phone"),
        (name, value) -> {
            if (value == null || value.length() < 7) return "***";
            return value.substring(0, 3) + "****" + value.substring(value.length() - 4);
        }
    );
}
```
