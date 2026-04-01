#  Springboot 集成 Consul 注册中心
{docsify-updated}

> https://docs.spring.io/spring-cloud-consul/docs/current/reference/html/

consul提供的各种语言的SDK/开发库： https://developer.hashicorp.com/consul/api-docs/libraries-and-sdks  
`spring-cloud-starter-consul-discovery` 使用的是  [consul-api](https://github.com/Ecwid/consul-api) ,依赖其底层的 `ConsulClient` 与 Consul 交互。

```
<project>
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>{spring-boot-version}</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
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
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

配置：
+ `spring.cloud.consul.discovery.enabled/spring.cloud.discovery.enabled` : 控制是否启用服务发现
+ `spring.cloud.consul.discovery.register` : 控制是否注册服务
+ `spring.cloud.consul.discovery.health-check-*` : 健康检查相关的
+ `spring.cloud.consul.discovery.register-health-check=false` ： 关闭健康检查
+ `spring.cloud.consul.discovery.heartbeat.*` : TTL 检查

### 自动配置类
```java
public class ConsulAutoServiceRegistrationAutoConfiguration {

    @Autowired
    AutoServiceRegistrationProperties autoServiceRegistrationProperties;

    @Bean
    @ConditionalOnMissingBean
    public ConsulAutoServiceRegistration consulAutoServiceRegistration(ConsulServiceRegistry registry,
          AutoServiceRegistrationProperties autoServiceRegistrationProperties, ConsulDiscoveryProperties properties,
          ConsulAutoRegistration consulRegistration) {
       return new ConsulAutoServiceRegistration(registry, autoServiceRegistrationProperties, properties,
             consulRegistration);
    }

    @Bean
    public ConsulAutoServiceRegistrationListener consulAutoServiceRegistrationListener(
          ConsulAutoServiceRegistration registration) {
       return new ConsulAutoServiceRegistrationListener(registration);
    }

    @Bean
    @ConditionalOnMissingBean
    public ConsulAutoRegistration consulRegistration(
          AutoServiceRegistrationProperties autoServiceRegistrationProperties, ConsulDiscoveryProperties properties,
          ApplicationContext applicationContext,
          ObjectProvider<List<ConsulRegistrationCustomizer>> registrationCustomizers,
          ObjectProvider<List<ConsulManagementRegistrationCustomizer>> managementRegistrationCustomizers,
          HeartbeatProperties heartbeatProperties) {
       return ConsulAutoRegistration.registration(autoServiceRegistrationProperties, properties, applicationContext,
             registrationCustomizers.getIfAvailable(), managementRegistrationCustomizers.getIfAvailable(),
             heartbeatProperties);
    }
```

#### ConsulDiscoveryProperties
`spring.cloud.consul.discovery` 前缀的配置对应的配置类是 `ConsulDiscoveryProperties`:
1. 如果要禁用服务发现（当你只作为服务提供者时），设置`spring.cloud.consul.discovery.enabled=false`，默认是 true
2. 如果要禁用服务注册（当你只作为服务消费者时），设置`spring.cloud.consul.discovery.register=fales`，默认是 true
3. 配置Consul健康检查，conusl发请求给服务以检查健康状态
   Consul 实例的健康检查默认位于"/actuator/health"，这是 Spring Boot Actuator 应用程序中健康端点的默认位置。如果使用非默认上下文路径或 servlet 路径（如 server.servletPath=/foo），即使是 Actuator 应用程序也需要更改。
4. 如果要禁用健康检查，设置 `spring.cloud.consul.discovery.register-health-check=false`,`management.health.consul.enabled=false`
5. TTL健康检查：应用程序会主动向 Consul 代理发送心跳信号，而不是 Consul 代理向应用程序发送请求。
6. 注册服务时提供 MetaData
7. `prefer-ip-address: true`: 注册时的地址信息使用IP，k8s环境下默认使用 pod 名字，会导致发现方调用失败

```
spring:
  application:
    name: trade-center-service
  cloud:
    service-registry:
      auto-registration:
        registerManagement: false //关闭management endpoint 的注册
    consul:
      host: consul
      port: 8500
      discovery:
        prefer-ip-address: true
        service-name: ${spring.application.name}
        register: true
        port: ${grpc.server.port}
        health-check-url: "http://${spring.cloud.consul.discovery.hostInfo.ipAddress}:${management.server.port}/actuator/health"
        health-check-interval: 10s  //健康检查的间隔
        health-check-critical-timeout: 1m  // 如果连续健康检查失败，并维持 critical 状态超过此时间（如 1 分钟），Consul 就会注销这个服务
        metadata:
             myfield: myvalue
             anotherfield: anothervalue
```
#### HeartbeatProperties
`HeartbeatProperties` 对应的配置前缀是 `spring.cloud.consul.discovery.heartbeat`。

```
spring:
  application:
    name: trade-center-service
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        hostInfo:
          ipAddress: localhost
        service-name: ${spring.application.name}
        register: true
        port: ${grpc.server.port}
        heartbeat:
          enabled: true
          ttl: 20s
          intervalRatio: 0.8
          reregisterServiceOnFailure: true
```

### 服务注册过程

#### 相关配置
`ConsulDiscoveryProperties`: springboot配置属性加载类,加载`spring.cloud.consul.discovery`前缀的配置，用于配置Consul服务发现与注册的配置

#### ConsulAutoRegistration
`ConsulAutoRegistration`: 封装需要注册的服务的信息，例如服务名、服务端口、健康检查、心跳等，它的 `registration` 方法会生成需要向 consul 注册的服务的信息（ `NewService` ）。

```java
public static ConsulAutoRegistration registration(
            AutoServiceRegistrationProperties autoServiceRegistrationProperties, ConsulDiscoveryProperties properties,
            ApplicationContext context, List<ConsulRegistrationCustomizer> registrationCustomizers,
            List<ConsulManagementRegistrationCustomizer> managementRegistrationCustomizers,
            HeartbeatProperties heartbeatProperties) {

      NewService service = new NewService();
      String appName = getAppName(properties, context.getEnvironment());
      service.setId(getInstanceId(properties, context));
      if (!properties.isPreferAgentAddress()) {
            service.setAddress(properties.getHostname());
      }
      service.setName(normalizeForDns(appName));
      service.setTags(new ArrayList<>(properties.getTags()));
      service.setEnableTagOverride(properties.getEnableTagOverride());
      service.setMeta(getMetadata(properties));

      if (properties.getPort() != null) {
            service.setPort(properties.getPort());
            // we know the port and can set the check
            setCheck(service, autoServiceRegistrationProperties, properties, context, heartbeatProperties);
      }

      ConsulAutoRegistration registration = new ConsulAutoRegistration(service, autoServiceRegistrationProperties,
                  properties, context, heartbeatProperties, managementRegistrationCustomizers);
      customize(registrationCustomizers, registration);
      return registration;
}
```

#### ConsulAutoServiceRegistration
`ConsulAutoServiceRegistration` : 封装服务注册过程的类，含有一个 `ConsulServiceRegistry` 属性，该类能真正使用consul sdk中的能力向 consul 注册服务

#### ConsulServiceRegistry
`ConsulServiceRegistry`： 真正与 consul 服务交互实施服务注册( `register`方法 )与卸载（`deregister`方法）的类。
```java
public class ConsulServiceRegistry implements ServiceRegistry<ConsulRegistration> {
      private final ConsulClient client;
      private final ConsulDiscoveryProperties properties;
      private final TtlScheduler ttlScheduler;
      private final HeartbeatProperties heartbeatProperties;

      public ConsulServiceRegistry(ConsulClient client, ConsulDiscoveryProperties properties, TtlScheduler ttlScheduler,
                  HeartbeatProperties heartbeatProperties) {
            this.client = client;
            this.properties = properties;
            this.ttlScheduler = ttlScheduler;
            this.heartbeatProperties = heartbeatProperties;
      }

      @Override
      public void register(ConsulRegistration reg) {
            log.info("Registering service with consul: " + reg.getService());
            try {
                  this.client.agentServiceRegister(reg.getService(), this.properties.getAclToken());
                  NewService service = reg.getService();
                  if (this.heartbeatProperties.isEnabled() && this.ttlScheduler != null && service.getCheck() != null
                              && service.getCheck().getTtl() != null) {
                        this.ttlScheduler.add(reg.getService());
                  }
            }
            catch (ConsulException e) {
                  if (this.properties.isFailFast()) {
                        log.error("Error registering service with consul: " + reg.getService(), e);
                        ReflectionUtils.rethrowRuntimeException(e);
                  }
                  log.warn("Failfast is false. Error registering service with consul: " + reg.getService(), e);
            }
      }

      @Override
      public void deregister(ConsulRegistration reg) {
            if (this.ttlScheduler != null) {
                  this.ttlScheduler.remove(reg.getInstanceId());
            }
            if (log.isInfoEnabled()) {
                  log.info("Deregistering service with consul: " + reg.getInstanceId());
            }
            this.client.agentServiceDeregister(reg.getInstanceId(), this.properties.getAclToken());
      }
}
```
`ConsulServiceRegistry` 注册服务时使用的 endpoint：
http://consul:8500/v1/agent/service/register?token=xxx

body:
```json
{
      "ID": "trade-center-service-8100",
      "Name": "trade-center-service",
      "Tags": [],
      "Address": "10.176.81.23",
      "Meta": {
            "secure": "false",
            "gRPC_port": "8101"
      },
      "Port": 8101,
      "Check": {
            "Interval": "10s",
            "HTTP": "http://10.176.81.23:8100/actuator/health",
            "Header": {},
            "DeregisterCriticalServiceAfter": "3m"
      }
}
```

#### ConsulAutoServiceRegistrationListener
`ConsulAutoServiceRegistrationListener`：服务注册的触发者
```java
public class ConsulAutoServiceRegistrationListener implements SmartApplicationListener {

      private final ConsulAutoServiceRegistration autoServiceRegistration;

      @Override
      public void onApplicationEvent(ApplicationEvent applicationEvent) {
            if (applicationEvent instanceof WebServerInitializedEvent) {
                  WebServerInitializedEvent event = (WebServerInitializedEvent) applicationEvent;

                  ApplicationContext context = event.getApplicationContext();
                  if (context instanceof ConfigurableWebServerApplicationContext) {
                        if ("management".equals(((ConfigurableWebServerApplicationContext) context).getServerNamespace())) {
                              return;
                        }
                  }
                  this.autoServiceRegistration.setPortIfNeeded(event.getWebServer().getPort());
                  this.autoServiceRegistration.start();
            }
      }
}
```
当收到Springboot的`WebServerInitializedEvent` 事件时，就会开始注册服务。这是 Spring-cloud-consul 的默认机制，如果没有 web 环境，就不会自动注册服务，此时就需要自行注册：
### 自定义注册-使用tcp health check

以下类可以实现两个功能：
1. 服务端不启用 Http 服务也能进行服务注册：通过监听`ApplicationStartedEvent`触发注册
2. 使用 tcp 作为健康检查

```
@Configuration
public class ConsulRegister implements ConsulRegistrationCustomizer{

    @Resource
    @Lazy
    ConsulAutoServiceRegistration registration;
    @Resource
    ConsulDiscoveryProperties properties;
    @Resource
    HeartbeatProperties ttlConfig;


    @EventListener
    public void pullUpConsulRegister(ApplicationStartedEvent event) {
        registration.start();
    }

    @Override
    public void customize(ConsulRegistration reg) {
        NewService service = reg.getService();
        Integer port = service.getPort();
        NewService.Check check = new NewService.Check();
        if (StringUtils.hasText(properties.getHealthCheckCriticalTimeout())) {
            check.setDeregisterCriticalServiceAfter(properties.getHealthCheckCriticalTimeout());
        }
        if (ttlConfig.isEnabled()) {
            // FIXME 3.0.0
            // https://github.com/spring-cloud/spring-cloud-consul/issues/614
            check.setTtl(ttlConfig.getTtl().getSeconds() + "s");
        }

        check.setTcp(service.getAddress().concat(":").concat(String.valueOf(port)));
        check.setGrpc("localhost:8101"); //或者设置 grpc
        check.setInterval(properties.getHealthCheckInterval());
        check.setTimeout(properties.getHealthCheckTimeout());
        check.setTlsSkipVerify(properties.getHealthCheckTlsSkipVerify());
        service.setCheck(check);
    }
}
```


### 服务发现

1. 使用 `Spring Cloud LoadBalancer`
Spring Cloud 支持 Feign（REST 客户端构建器）和 Spring RestTemplate，以便使用逻辑服务名称/ID 而不是物理 URL 查找服务。Feign 和具有 RestTemplate 都利用 Spring Cloud LoadBalancer 实现客户端负载平衡。
1. 使用 `DiscoveryClient`
   ```
   @Autowired
   private DiscoveryClient discoveryClient;

   public String serviceUrl() {
       List<ServiceInstance> list = discoveryClient.getInstances("STORES");
       if (list != null && list.size() > 0 ) {
           return list.get(0).getUri();
       }
       return null;
   }
   ```

服务发现使用的 `HealthConsulClient` 的方法：
```java
public Response<List<HealthService>> getHealthServices(String serviceName, HealthServicesRequest healthServicesRequest) {
      HttpResponse httpResponse = this.rawClient.makeGetRequest("/v1/health/service/" + serviceName, healthServicesRequest.asUrlParameters());
      if (httpResponse.getStatusCode() == 200) {
      List<HealthService> value = (List)GsonFactory.getGson().fromJson(httpResponse.getContent(), (new TypeToken<List<HealthService>>() {
      }).getType());
      return new Response(value, httpResponse);
      } else {
      throw new OperationException(httpResponse);
      }
}
```

调用的 API 是 `/v1/health/service/` .


### 重启consul 后，服务不能重新注册：
https://github.com/spring-cloud/spring-cloud-consul/issues/197  
https://github.com/spring-cloud/spring-cloud-consul/pull/691  
https://github.com/spring-cloud/spring-cloud-consul/issues/727  

```
Resolved.
need to add the below 2 properties:
spring.cloud.consul.discovery.heartbeat.enabled= true
spring.cloud.consul.discovery.heartbeat.reregister-service-on-failure=true
```

原理：  
启用了 `spring.cloud.consul.discovery.heartbeat.enabled= true` 后，服务注册时会启动一个定时任务：
```
if (this.heartbeatProperties.isEnabled() && this.ttlScheduler != null && service.getCheck() != null
            && service.getCheck().getTtl() != null) {
      this.ttlScheduler.add(reg.getService());
}
```

`ConsulHeartbeatAutoConfiguration` 会注册一个bean `TtlScheduler` -> `ConsulHeartbeatTask`
```
@Override
public void run() {
      try {
            this.ttlScheduler.client.agentCheckPass(this.checkId);
            if (log.isDebugEnabled()) {
                  log.debug("Sending consul heartbeat for: " + this.checkId);
            }
      }
      catch (OperationException e) {
            if (this.ttlScheduler.heartbeatProperties.isReregisterServiceOnFailure()
                        && this.ttlScheduler.reregistrationPredicate.isEligible(e)) {
                  log.warn(e.getMessage());
                  NewService registeredService = this.ttlScheduler.registeredServices.get(this.serviceId);
                  if (registeredService != null) {
                        if (log.isInfoEnabled()) {
                              log.info("Re-register " + registeredService);
                        }
                        this.ttlScheduler.client.agentServiceRegister(registeredService,
                                    this.ttlScheduler.discoveryProperties.getAclToken());
                  }
                  else {
                        log.warn("The service to re-register is not found.");
                  }
            }
            else {
                  throw e;
            }
      }
}
```
这个 task 会调用 `/agent/check/pass/:check_id` API 触发服务检查
注意：以上解决方案只有在 consul 集群支持持久化存储状态的时候生效。当以 -dev 模式运行时，会产生 checkId 不存在的 404 错误，因为consul重启后所有的checks数据丢失，所以`this.ttlScheduler.client.agentCheckPass(this.checkId);`会报错。这种情况下，除了配置`spring.cloud.consul.discovery.heartbeat.reregister-service-on-failure=true`外，还要注册以下bean:
```java
@Configuration
public class ConsulConfiguration {
      @Bean
      public ReregistrationPredicate test(){
            return e-> Objects.nonNull(e);
      }
}
```
这样才会使服务重新注册service。
默认注册的 `ReregistrationPredicate` 只有在 500 返回码时才会激活重新注册：
```
@Bean
@ConditionalOnMissingBean
public ReregistrationPredicate reRegistrationPredicate() {
      return ReregistrationPredicate.DEFAULT;
}

public interface ReregistrationPredicate {

/**
      * test if the exception is eligible for re-registration.
      * @param e OperationException
      * @return if the exception is eligible for re-registration
      */
boolean isEligible(OperationException e);

/**
      * Default implementation that performs re-registration when the status code is 500.
      */
ReregistrationPredicate DEFAULT = e -> e.getStatusCode() == 500;
}
```

### 问题
1. preferIpAddress ： 测试环境需要IP访问，主机名不通
2. critical状态的service 不会自动取消注册：
      `spring.cloud.consul.discovery.health-check-critical-timeout: 1m` - Timeout to deregister services critical for longer than timeout。
