# Spring Cloud Gateway 原理
{docsify-updated}

## 基础概念与原理
+ `Route` ： 网关的基本构建单元。由ID、目标URI、 `Predicate` 集合和 `filters` 集合定义。当 `Predicate` 为真时，路由即匹配成功。
+ `Predicate` :  Java 8 `Predicate` 接口。输入类型为Spring框架的 `ServerWebExchange` 。 可以匹配HTTP请求中的任何内容，例如请求头或参数。
+ `Filter` : 这些是通过特定工厂构造的 `GatewayFilter` 实例。可以在发送请求到下游之前或之后修改请求和响应。
+ `RouteDefinitionLocator` : 根据定义（yml/property文件/consul 配置）加载路由配置信息。
+ `RouteLocator` : 根据配置信息加载路由。

<center><img src="pics/spring_cloud_gateway_diagram.png" alt=""></center>

客户端向Spring Cloud网关发送请求。若网关处理器映射判定请求匹配路由，则将其转发至网关Web处理器。该处理器通过专属过滤器链处理请求。过滤器以点分隔线划分的原因在于：它们可在代理请求发送前、后执行逻辑。所有"预处理"过滤器逻辑执行完毕后，代理请求才会发出。代理请求发出后，将执行"后置"过滤器逻辑。

`DispatcherHandler` 
<center><img src="pics/dispatcher.png" alt=""></center>

```
┌────────────────────────┐
│ Client（外部请求）     │
└─────────────┬──────────┘
              ▼
     ┌────────────────────┐
     │ RoutePredicateHandlerMapping │  ← 匹配 Route
     └─────────────┬───────────────┘
                   ▼
     ┌────────────────────┐
     │ FilteringWebHandler │ ← 过滤器链执行器
     └─────────────┬───────────────┘
                   ▼
     ┌────────────────────┐
     │ GatewayFilterChain │ ← 逐层执行过滤器
     └─────────────┬───────────────┘
                   ▼
     ┌────────────────────┐
     │ Netty HttpClient    │ ← 转发请求到目标服务（Reactive）
     └─────────────┬───────────────┘
                   ▼
     ┌────────────────────┐
     │ Response 返回 Client │
     └────────────────────┘
```

> https://zhuanlan.zhihu.com/p/11306458448

<center><img src="pics/route.png" width="60%"></center>


## 自动配置原理
```
GatewayAutoConfiguration
RouteDefinitionRouteLocator
CachingRouteLocator
```

## Consul 动态配置路由
```
@Bean
@RefreshScope
@ConfigurationProperties("spring.cloud.gateway") //使用最新配置的一定要去除这行
@Primary
public GatewayProperties refreshableGatewayProperties() {
     GatewayProperties gatewayProperties = new GatewayProperties();
     return gatewayProperties;
}

spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
          - id: abc
            uri: https://www.abc.com
            predicates:
            - Path=/abc
```

动态刷新核心：RefreshRoutesEvent 事件机制

`CachingRouteLocator` 实现了 Spring 的 `ApplicationListener` 接口，它监听一个特殊的事件： `RefreshRoutesEvent` 。
无论何时，只要 Spring 应用上下文中发布了 `RefreshRoutesEvent` 事件， `CachingRouteLocator` 就会执行以下操作：
1. 清除缓存： 立即清除内部存储的路由列表缓存。
2. 重新加载： 再次调用底层的 `RouteDefinitionLocator` ，从最新的 Spring Environment (`GatewayProperties`)中获取新的路由配置。
3. 更新路由： 将最新的路由列表加载到 Gateway 的路由表中，实现热更新。

这里第2步就是关键，我们只要将 `GatewayProperties` 声明为 `RefreshScope` 这样就能实时更新配置值， `RouteDefinitionLocator` 就能加载到最新的配置路由了。这里有两个小坑：
1. Gateway 自动配置中会配置 `GatewayProperties` ，所以我们必须加上 `@Primary` 覆盖自动配置的 bean
2. Gateway 最新的路径配置前缀是 `spring.cloud.gateway.server.webflux` 不是文档中例子中的 `spring.cloud.gateway`， 为了兼容老的配置使用了 `GatewayServerWebfluxPropertiesMigrationListener` 来兼容了。所以如果使用老配置的，要加上 `@ConfigurationProperties("spring.cloud.gateway")` 注解