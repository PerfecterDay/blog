# API 版本化支持
{docsify-updated}

> https://spring.io/blog/2025/09/16/api-versioning-in-spring  
> https://docs.spring.io/spring-framework/reference/7.0-SNAPSHOT/web/webmvc-versioning.html


## ApiVersionStrategy
```
default @Nullable Comparable<?> resolveParseAndValidateVersion(HttpServletRequest request) {
    String value = resolveVersion(request);
    Comparable<?> version;
    if (value == null) {
        version = getDefaultVersion();
    }
    else {
        try {
            version = parseVersion(value);
        }
        catch (Exception ex) {
            throw new InvalidApiVersionException(value, null, ex);
        }
    }
    validateVersion(version, request);
    return version;
}
```

## ApiVersionResolver
`PathApiVersionResolver` 解析路径中的版本参数字符串信息

## ApiVersionParser
`SemanticApiVersionParser` 将字符串解析为 Version 对象

## Validation

## 配置
```
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer
                // 使用 URL Path 方式
                .usePathSegment(0);
    }

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.addPathPrefix("/{version}", HandlerTypePredicate.forAnnotation(RestController.class));
    }
}
```

也可以在配置文件中配置：
```
spring:
  mvc:
    apiversion:
      default: v1.0
      use:
        header: api-version
        path-segment: 0
```

## 问题
在服务端 `Controller` 内不能使用 `@GetExchange/@PostExchange` 等注解代替 `@GetMapping/@PostMapping` 等注解。 否则版本信息不会被扫描到。