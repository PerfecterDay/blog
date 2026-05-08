# Springboot 优雅日志打印库-logbook
{docsify-updated}

> https://github.com/zalando/logbook

```
DefaultLogbook
DefaultLogbookFactory

HttpLogFormatter
JacksonJsonFieldBodyFilter
```

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
