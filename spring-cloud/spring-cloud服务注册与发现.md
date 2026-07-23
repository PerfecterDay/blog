# Spring Cloud 服务注册与发现
{docsify-updated}

## 服务信息抽象
`ServiceInstance` 在服务发现系统里边代表了一个服务实例信息，是一个接口，主要抽象了服务实例的id、服务id、host、port、uri、metadata等信息。
```
public interface ServiceInstance {
	default @Nullable String getInstanceId() {
		return null;
	}

	String getServiceId();

	String getHost();

	int getPort();

	boolean isSecure();

	URI getUri();

	@Nullable Map<String, String> getMetadata();

	default @Nullable String getScheme() {
		return null;
	}

	static URI createUri(ServiceInstance instance) {
		String scheme = (instance.isSecure()) ? "https" : "http";
		int port = instance.getPort();
		if (port <= 0) {
			port = (instance.isSecure()) ? 443 : 80;
		}
		String uri = String.format("%s://%s:%s", scheme, instance.getHost(), port);
		return URI.create(uri);
	}
}
```

`DefaultServiceInstance` 是 `ServiceInstance` 的默认实现，主要通过构造函数传入服务实例id、服务id、host、port、uri、metadata等信息。

`org.springframework.cloud.client.serviceregistry.Registration` 是 spring-cloud-commons 提供的服务注册信息的抽象，也继承自 `ServiceInstance` 接口。
```
public interface Registration extends ServiceInstance {
}
```
各个服务发现框架通过实现 `Registration` 接口提供各自的服务注册信息的实现。比如：
+ spring-cloud-consul-discovery 中的 `ConsulRegistration` 就是实现的改接口提供了自己的实现。
+ spring-cloud-starter-alibaba-nacos-discovery 中的 `NacosRegistration` 提供了 Nacos 的实现。

## 服务注册
`ServiceRegistry` 接口提供了服务注册与注销的方法：
```
public interface ServiceRegistry<R extends Registration> {
	void register(R registration);

	void deregister(R registration);

	void close();

	void setStatus(R registration, String status);

	<T> T getStatus(R registration);
}
```
各个服务发现框架能提供各自的实现，比如 `NacosServiceRegistry` 和 `ConsulServiceRegistry` 等。

`AutoServiceRegistration` 自动服务注册的接口定义：
```
public interface AutoServiceRegistration {
}
```

`AbstractAutoServiceRegistration` 提供了自动服务注册过程中需要用到的一些基础功能，各个服务发现框架可以继承该类来实现自动服务注册功能：
+ `ConsulAutoServiceRegistration` 
+ `NacosAutoServiceRegistration`


`AutoServiceRegistration/AbstractAutoServiceRegistration` 主要为了支持微服务在启动时自动注册到服务注册中心，在关闭时自动注销。

## 服务发现-DiscoveryClient
Spring cloud 使用接口 `DiscoveryClient` 来抽象服务发现客户端，各服务发现框架提供自己的实现。
```
public interface DiscoveryClient extends Ordered {
	int DEFAULT_ORDER = 0;

	String description();

	List<ServiceInstance> getInstances(String serviceId);

	List<String> getServices();

	default void probe() {
		getServices();
	}

	@Override
	default int getOrder() {
		return DEFAULT_ORDER;
	}
}
```
`DiscoveryClient` 中和新方法 `List<ServiceInstance> getInstances(String serviceId);` 提供了根据服务ID获取 `ServiceInstance` 列表的方法。

为 Consul 提供的实现 `ConsulDiscoveryClient`， 为 Nacos 提供的实现 `NacosServiceDiscovery`。

Spring cloud 默认的实现 `SimpleDiscoveryClient`， 通过配置属性获取服务实例信息：`spring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080` ，其中 `spring.cloud.discovery.client.simple.instances` 是通用前缀， `service1` 代表相关服务的 ID，`[0]` 表示实例的索引号（如示例所示，索引从 0 开始），而 `uri` 的值则是该实例可用的实际 `URI` 。