# SPring cloud gateway
{docsify-updated}

> https://docs.spring.io/spring-cloud-gateway/reference/

## 集成
```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-cloud.version>2025.0.0</spring-cloud.version>
</properties>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webmvc</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-circuitbreaker-resilience4j</artifactId>
</dependency>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```



## Predicate 配置
> https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/request-predicates-factories.html

## Filter 配置
> https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories.html


## TLS 和 SSL
配置 Gateway 服务本身以 SSL 方式启动，这意味着客户端需要 HTTPS 才能连接到 Gateway：
```
server:
  ssl:
    enabled: true
    key-alias: scg
    key-store-password: scg1234
    key-store: classpath:scg-keystore.p12
    key-store-type: PKCS12
```

配置 Gateway 以 HTTPS 去转发流量到上游服务时：
```
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trustedX509Certificates:
          - cert1.pem
          - cert2.pem
```

## HttpClientCustomizer
```
import org.springframework.cloud.gateway.config.HttpClientCustomizer;
import reactor.netty.http.client.HttpClient;

public class MyHttpClientCustomizer implements HttpClientCustomizer {

    @Override
    public HttpClient customize(HttpClient httpClient) {
        // Customize the HTTP client here
        return httpClient.tcpConfiguration(tcpClient -> {
            // Set the connect timeout to 5 seconds
            tcpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
            // Set the read timeout to 10 seconds
            tcpClient.option(ChannelOption.SO_TIMEOUT, 10000);
            return tcpClient;
        });
    }
}

@Configuration
public class GatewayConfiguration {

    @Bean
    public HttpClientCustomizer myHttpClientCustomizer() {
        return new MyHttpClientCustomizer();
    }
}
```

## Metadata
```
spring:
  cloud:
    gateway:
      routes:
      - id: route_with_metadata
        uri: https://example.org
        metadata:
          optionName: "OptionValue"
          compositeObject:
            name: "value"
          iAmNumber: 1


Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
// get all metadata properties
route.getMetadata();
// get a single metadata property
route.getMetadata(someKey);
```

## 超时设置
全局超时设置：
```
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000
        response-timeout: 5s
```

不同路由设置：
```
- id: per_route_timeouts
    uri: https://example.org
    predicates:
    - name: Path
        args:
        pattern: /delay/{timeout}
    metadata:
    response-timeout: 200
    connect-timeout: 200
```

## CORS 配置
全局配置：
```
spring:
  cloud:
    gateway:
      globalcors:
        add-to-simple-url-handler-mapping: true
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://docs.spring.io"
            allowedMethods:
            - GET
```

不同路由设置：
```
spring:
  cloud:
    gateway:
      routes:
      - id: cors_route
        uri: https://example.org
        predicates:
        - Path=/service/**
        metadata:
          cors:
            allowedOrigins: '*'
            allowedMethods:
              - GET
              - POST
            allowedHeaders: '*'
            maxAge: 30
```

## 服务发现路由
可以配置网关，使其基于在与 `DiscoveryClient` 兼容的服务注册表中注册的服务创建路由。

默认情况下，创建的路由使用协议 `lb://service-name`（其中 `service-name` 是 `DiscoveryClient::getServices` 方法返回的字符串），这意味着它们会进行负载均衡。因此，还需引入 `org.springframework.cloud:spring-cloud-starter-loadbalancer` 依赖项，确保其位于类路径中。

要启用此功能，请设置 `spring.cloud.gateway.discovery.locator.enabled=true` ，并确保 `DiscoveryClient` 实现（如 Netflix Eureka、Consul、Zookeeper 或 Kubernetes）位于类路径且处于启用状态。

### 原理
启用 `discovery.locator.enabled=true` 后，Gateway 会注册一个特殊的组件：`DiscoveryClientRouteDefinitionLocator`.
它会：
1. 从注册中心获取所有服务实例（通过 `DiscoveryClient` ）；
2. 为每个服务自动生成一个 `RouteDefinition` ；
3. Route 的目标 URI 使用 `lb://service-name`；
4. 交由 `LoadBalancerClientFilter` 负责选择具体实例。

如：
```
spring:
  cloud:
    gateway:lo
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/user-service/**
```

我们也可以手动配置路由：
```
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: user
          uri: lb://user-service
          predicates:
            - Path=/user/**
```

## Actuator
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

management:
  endpoint:
    gateway:
      access: read-only

  endpoints:
    web:
      exposure:
        include: gateway
```
http://localhost:8080/actuator/gateway