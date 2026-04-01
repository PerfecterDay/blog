# Spring Cloud Commons
{docsify-updated}

> https://docs.spring.io/spring-cloud-commons/reference/index.html


`Spring Cloud Context` 为 Spring Cloud 应用程序的 ApplicationContext 提供实用工具和特殊服务（bootstrap context, encryption, refresh scope, and environment endpoints）。 `Spring Cloud Commons` 是一组抽象层和通用类，用于不同 Spring Cloud 实现（如 Spring Cloud Netflix 和 Spring Cloud Consul）。

## Spring Cloud Context
Spring Boot 对如何使用 Spring 构建应用程序有着明确的见解。例如，它为常见配置文件提供了约定位置，并为常规管理和监控任务提供了端点。Spring Cloud 在此基础上构建，并添加了若干功能，这些功能是系统中许多组件会使用或偶尔需要的。

### Environment Changes
应用程序监听 `EnvironmentChangeEvent` 事件，并通过几种标准方式响应环境变更（如使用 `@Bean` 添加一个 `ApplicationListeners`）。当检测到 `EnvironmentChangeEvent` 时，该事件包含已变更的键值列表，应用程序利用这些信息来：
+ 重新绑定上下文中的任何 `@ConfigurationProperties` bean。
+ 为 `logging.level.*` 下的任何属性设置日志记录级别。

请注意，Spring Cloud Config Client 默认不会轮询 `Environment` 中的变更。通常我们不建议采用这种方式检测变更（尽管您可以通过 `@Scheduled` 注解进行配置）。若您拥有分布式的应用实例，更优方案是向所有实例广播 `EnvironmentChangeEvent` 事件，而非让它们轮询变更（例如通过 Spring Cloud Bus 实现）。

`EnvironmentChangeEvent` 涵盖了大量的属性更新应用场景，只要您能够实际修改 `Environment` 并发布 `EnvironmentChangeEvent` 事件即可。请注意这些 API 是公开的，且属于 Spring 核心组件。然后就可通过访问 `/configprops` 端点（Spring Boot Actuator 的标准功能）验证变更是否已绑定至 `@ConfigurationProperties` bean。例如， `DataSource` 的 `maxPoolSize` 可在运行时动态调整（Spring Boot 默认创建的 DataSource 即为 `@ConfigurationProperties` Bean），从而实现容量动态扩展。

测试代码：
```
@Resource
ConfigurableEnvironment environment;

@Resource
ApplicationContext applicationContext;

@GetMapping("/aaa")
public String aaa() {
    Map<String, Object> map = new HashMap<>();
    map.put("cms.url", "new-value"); // 修改或新增属性

    // 添加到最前面，优先级最高
    MapPropertySource propertySource = new MapPropertySource("dynamic-properties", map);
    environment.getPropertySources().addFirst(propertySource);

    HashSet set = new HashSet();
    set.add("cms.url");
    applicationContext.publishEvent(new EnvironmentChangeEvent(set));
    return environment.getProperty("cms.url");
}
```

访问 `http://localhost:8091/actuator/configprops` 可以看到 new-value 的值。

但是重新绑定 `@ConfigurationProperties` 无法覆盖另一类重要用例场景：当您需要更精细地控制刷新机制，或要求变更在整个 `ApplicationContext` 中具有原子性时。

例如上边的修改 `cms.url` 的例子，当事件发布后，`environment.getProperty("cms.url")` 代码和 `@ConfigurationProperties` 注解的配置 bean 都会得到新值。但是，如果有个bean 定义：
```
@Component
public class MyService {
    @Value("${cms.url}")
    private String url;
}
```
此时的 url 不会更新成新值，因为 @Value 的注入是在 Bean 创建时完成的，Spring 不会因为 Environment 变化就重新注入 Bean 的字段。

为解决这些需求，我们引入了 `@RefreshScope` 注解。

## Spring Cloud Commons
诸如服务发现、负载均衡和熔断限流等模式，天然适合构建一个通用抽象层，该层可被所有Spring Cloud客户端使用，且与具体实现无关。（比如使用 Eurika 或者 Consul 都可以实现服务发现）。

### 服务注册与发现

#### 服务发现客户端
`@EnableDiscoveryClient` 会扫描注册 `DiscoveryClient` 的实现。

Spring Cloud 默认同时提供阻塞式和响应式服务发现客户端。可通过设置 `spring.cloud.discovery.blocking.enabled=false` 或 `spring.cloud.discovery.reactive.enabled=false` 轻松禁用阻塞式和/或响应式客户端。若需完全禁用服务发现功能，只需将 `spring.cloud.discovery.enabled` 设置为 `false` 即可。

##### 优先级设置
`spring.cloud.{clientIdentifier}.discovery.order` 属性可用于设置优先级。比如 `spring.cloud.eureka.client.order=0` ，`spring.cloud.consul.discovery.order=1` ，则 `Consul` 优先级更高。

##### SimpleDiscoveryClient
如果类路径中不存在由服务注册表支持的 `DiscoveryClient` ，则将使用 `SimpleDiscoveryClient` 实例，该实例通过属性获取服务和实例的相关信息。

可用实例的相关信息应通过以下格式的属性传递： `spring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080` ，其中 `spring.cloud.discovery.client.simple.instances` 是通用前缀， `service1` 代表目标服务的ID， `[0]`表示实例的索引号（如示例所示，索引从0开始），而uri的值即为该实例实际可用的URI地址。

#### 服务注册
Commons 现提供 `ServiceRegistry` 接口，该接口包含 `register(Registration)` 和 `deregister(Registration)` 等方法，允许您提供自定义注册服务。 

`Registration` 是一个标记接口。每个 `ServiceRegistry` 实现都拥有其专属的 `Registration` 实现:
+ `ZookeeperRegistration` 用于配合 `ZookeeperServiceRegistry` 使用
+ `EurekaRegistration` 用于配合 `EurekaServiceRegistry` 使用
+ `ConsulRegistration` 用于配合 `ConsulServiceRegistry` 使用

若正在使用 `ServiceRegistry` 接口，则需为所使用的 `ServiceRegistry` 实现传递正确的 `Registration` 实现。

默认情况下，服务会自动注册， 通过设置 `spring.cloud.service-registry.auto-registration.enabled=false` 可以禁用自动注册。 `@EnableDiscoveryClient(autoRegister=false)` 也可禁用自动注册。

当服务自动注册时，将触发两个事件：
+ 第一个事件名为 `InstancePreRegisteredEvent` ，在服务注册前触发；
+ 第二个事件名为 `InstanceRegisteredEvent` ，在服务注册后触发。

我们可以注册 `ApplicationListener` 监听器来监听并响应这些事件。




