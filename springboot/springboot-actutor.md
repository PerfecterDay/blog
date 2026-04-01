#  Springboot Actuator
{docsify-updated}

> [docs](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)

Springboot Actuator 可以暴露一些 http 端点让用户能查看 springboot 应用的运行信息。默认的访问 url 是 `/actuator/{endpoint}` 。
默认情况下，除 `shutdown` 端点外的所有端点都已启用。要配置端点的启用，请使用其 `management.endpoint.<id>.enabled` 属性。

添加下面的依赖就能开启 Springboot Actuator：
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

一个端点如果想要通过Http或者Jmx 访问，必需 `enable` 它且 `expose` 出来。从 springboot 2.x 版本开始，默认只暴露 /health 端点。如果需要开启其它端点，需要增加下面配置：
```
management:
  endpoints:
    web:
	  base-path: /mgmt
      exposure:
        include: "*"
        exclude: "env"
	jmx:
      exposure:
        include: "health,info"
```

可以通过 url : http://localhost:8050/user/mgmt/，(默认是http://localhost:8050/actuator/ 如果设置了 `context-path`，需要加上，如 http://localhost:8050/user/mgmt/) ，这个endpoint 可以获取到所有可访问的端点，示例如下：
```
{
	"_links": {
		"self": {
			"href": "http://localhost:8050/actuator",
			"templated": false
		},
		"consul": {
			"href": "http://localhost:8050/actuator/consul",
			"templated": false
		},
		"features": {
			"href": "http://localhost:8050/actuator/features",
			"templated": false
		},
		"caches": {
			"href": "http://localhost:8050/actuator/caches",
			"templated": false
		},
		"caches-cache": {
			"href": "http://localhost:8050/actuator/caches/{cache}",
			"templated": true
		},
		"health": {
			"href": "http://localhost:8050/actuator/health",
			"templated": false
		},
		"health-path": {
			"href": "http://localhost:8050/actuator/health/{*path}",
			"templated": true
		},
		"info": {
			"href": "http://localhost:8050/actuator/info",
			"templated": false
		},
		"conditions": {
			"href": "http://localhost:8050/actuator/conditions",
			"templated": false
		},
		"configprops": {
			"href": "http://localhost:8050/actuator/configprops",
			"templated": false
		},
		"configprops-prefix": {
			"href": "http://localhost:8050/actuator/configprops/{prefix}",
			"templated": true
		},
		"loggers-name": {
			"href": "http://localhost:8050/actuator/loggers/{name}",
			"templated": true
		},
		"loggers": {
			"href": "http://localhost:8050/actuator/loggers",
			"templated": false
		},
		"heapdump": {
			"href": "http://localhost:8050/actuator/heapdump",
			"templated": false
		},
		"threaddump": {
			"href": "http://localhost:8050/actuator/threaddump",
			"templated": false
		},
		"metrics-requiredMetricName": {
			"href": "http://localhost:8050/actuator/metrics/{requiredMetricName}",
			"templated": true
		},
		"metrics": {
			"href": "http://localhost:8050/actuator/metrics",
			"templated": false
		},
		"scheduledtasks": {
			"href": "http://localhost:8050/actuator/scheduledtasks",
			"templated": false
		},
		"mappings": {
			"href": "http://localhost:8050/actuator/mappings",
			"templated": false
		},
		"refresh": {
			"href": "http://localhost:8050/actuator/refresh",
			"templated": false
		},
		"serviceregistry": {
			"href": "http://localhost:8050/actuator/serviceregistry",
			"templated": false
		}
	}
}
```

如果要开启 httptrace endpoint，需要注入一个 `HttpTraceRepository` 类型的 bean。具体原因如下 `@ConditionalOnBean({HttpTraceRepository.class})`：
```
@AutoConfiguration
@ConditionalOnWebApplication
@ConditionalOnProperty(
    prefix = "management.trace.http",
    name = {"enabled"},
    matchIfMissing = true
)
@ConditionalOnBean({HttpTraceRepository.class})
@EnableConfigurationProperties({HttpTraceProperties.class})
public class HttpTraceAutoConfiguration {
    public HttpTraceAutoConfiguration() {
    }
```

Actuator 的配置在 `org.springframework.boot.actuate.autoconfigure...` 配置包下都能找到。 


## 自定义健康检查逻辑
```
@Component
public class CustomHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // 举例：当本地有一个文件存在，则视为不健康
        if (Files.exists(Paths.get("/tmp/block-envoy"))) {
            return Health.down()
                .withDetail("reason", "manual offline via file")
                .build();
        }

        // 可以加更多自定义逻辑，比如数据库、缓存等探测

        return Health.up().withDetail("status", "everything is fine").build();
    }
}
```

## 原理
当配置了 management.port 与 server.port 不一致时， actuator 会另外启动一个独立的 WebServer 来处理 actuator 相关的请求，不会影响主业务流量端口。
```
ManagementContextAutoConfiguration
       ↓
ManagementContext (child ApplicationContext)
       ↓
ManagementWebServerFactory
       ↓
Management WebServer (Tomcat/Netty)
       ↓
ServletWebServerApplicationContext
       ↓
DispatcherServletAutoConfiguration
       ↓
DispatcherServlet
```

## 为什么建议生产环境分开端口
1. 安全隔离
Actuator 有很多敏感端点：
+ /actuator/env → 全量配置，可能有密码
+ /actuator/beans → 所有 bean
+ /actuator/heapdump → JVM 堆转储（非常敏感）
+ /actuator/threaddump

management 端口可以单独建防火墙规则、只允许内网、只允许 k8s 探针访问，业务端口对外开放，管理端口不对外，安全性提升非常大。

2. 服务自愈体系依赖 Actuator 独立可用 （K8s/Consul/Nacos）
异常情况下，业务端口会不响应，但 Actuator 端口仍能正常响应 → 让 K8s 正常探测、重启容器，能极大提升服务自愈能力。

3. 业务负载不会影响管理能力
在高峰期：
+ 业务端口 QPS 上万
+ Actuator 的 metrics/health 等运维请求无法打进来

如果两个端口独立：
+ 业务流量再大，管理端口照样能访问（因为是独立 WebServer）
+ Prometheus 拉 metrics 也不会超时

监控更可靠。
