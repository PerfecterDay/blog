# Skywalking
{docsify-updated}

- [Skywalking](#skywalking)
	- [基础概念-OpenTracing](#基础概念-opentracing)
	- [基础框架](#基础框架)
	- [skywalking-oap docker](#skywalking-oap-docker)
	- [skywalking-ui docker](#skywalking-ui-docker)
	- [Skywalking agent](#skywalking-agent)

### 基础概念-OpenTracing
OpenTracing中的Trace（调用链）通过归属于此调用链的Span来隐性的定义。 特别说明，一条Trace（调用链）可以被认为是一个由多个Span组成的有向无环图（DAG图）， Span与Span的关系被命名为References。

译者注: Span，可以被翻译为跨度，可以被理解为一次方法调用, 一个程序块的调用, 或者一次RPC/数据库访问.只要是一个具有完整时间周期的程序访问，都可以被认为是一个span.在此译本中，为了便于理解，Span和其他标准内声明的词汇，全部不做名词翻译。

例如：下面的示例Trace就是由8个Span组成：
```
单个Trace中，span间的因果关系
        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C 是 Span A 的孩子节点, ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G 在 Span F 后被调用, FollowsFrom)

```

**Span间关系**
一个Span可以与一个或者多个SpanContexts存在因果关系。OpenTracing目前定义了两种关系：ChildOf（父子） 和 FollowsFrom（跟随）。这两种关系明确的给出了两个父子关系的Span的因果模型。 将来，OpenTracing可能提供非因果关系的span间关系。（例如：span被批量处理，span被阻塞在同一个队列中，等等）。

1. ChildOf 引用: 一个span可能是一个父级span的孩子，即"ChildOf"关系。在"ChildOf"引用关系下，父级span某种程度上取决于子span。下面这些情况会构成"ChildOf"关系：
	一个RPC调用的服务端的span，和RPC服务客户端的span构成ChildOf关系
	一个sql insert操作的span，和ORM的save方法的span构成ChildOf关系
	很多span可以并行工作（或者分布式工作）都可能是一个父级的span的子项，他会合并所有子span的执行结果，并在指定期限内返回

2. FollowsFrom 引用: 一些父级节点不以任何方式依赖他们子节点的执行结果，这种情况下，我们说这些子span和父span之间是"FollowsFrom"的因果关系。"FollowsFrom"关系可以被分为很多不同的子类型，未来版本的OpenTracing中将正式的区分这些类型	.



### 基础框架
SkyWalking 逻辑上分为四部分: 探针, 平台后端, 存储和用户界面.
<center><img src="pics/skywalking-framework.png" width="70%"></center>

+ 探针 基于不同的来源可能是不一样的, 但作用都是收集数据, 将数据格式化为 SkyWalking 适用的格式.
+ 平台后端, 支持数据聚合, 数据分析以及驱动数据流从探针到用户界面的流程。分析包括 Skywalking 原生追踪和性能指标以及第三方来源，包括 Istio 及 Envoy telemetry , Zipkin 追踪格式化等。 你甚至可以使用 Observability Analysis Language 对原生度量指标 和 用于扩展度量的计量系统 自定义聚合分析。
+ 存储 通过开放的插件化的接口存放 SkyWalking 数据. 你可以选择一个既有的存储系统, 如 ElasticSearch, H2 或 MySQL 集群(Sharding-Sphere 管理),也可以选择自己实现一个存储系统. 当然, 我们非常欢迎你贡献新的存储系统实现。
+ UI 一个基于接口高度定制化的Web系统，用户可以可视化查看和管理 SkyWalking 数据


<center><img src="pics/skywalking-fram.jpg" width="70%"></center>
整个架构，分成上、下、左、右四部分：

>考虑到让描述更简单，我们舍弃掉 Metric 指标相关，而着重在 Tracing 链路相关功能。

+ 上部分 Agent ：负责从应用中，收集链路信息，发送给 SkyWalking OAP 服务器。目前支持 SkyWalking、Zikpin、Jaeger 等提供的 Tracing 数据信息。而我们目前采用的是，SkyWalking Agent 收集 SkyWalking Tracing 数据，传递给服务器。
+ 下部分 SkyWalking OAP ：负责接收 Agent 发送的 Tracing 数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。
+ 右部分 Storage ：Tracing 数据存储。目前支持 ES、MySQL、Sharding Sphere、TiDB、H2 多种存储器。而我们目前采用的是 ES ，主要考虑是 SkyWalking 开发团队自己的生产环境采用 ES 为主。
+ 左部分 SkyWalking UI ：负责提供控台，查看链路等等。

### skywalking-oap docker

```
FROM --platform=linux/amd64 apache/skywalking-oap-server:9.2.0

ENV SW_ES_USER elastic
ENV SW_ES_PASSWORD xz3H7rrPCCxrbsdt
ENV SW_STORAGE_ES_CLUSTER_NODES es-sg-vfp331pa70007h94i.elasticsearch.aliyuncs.com:9200
ENV SW_STORAGE_ES_HTTP_PROTOCOL http
```

启动镜像时的 ES 环境变量设置：
https://skywalking.apache.org/docs/main/next/en/setup/backend/backend-storage/ 
https://hub.docker.com/r/apache/skywalking-oap-server

OAP-server 的端口：
```
core:
  default:
    restHost: 0.0.0.0
    restPort: 12800
    restContextPath: /
    gRPCHost: 0.0.0.0
    gRPCPort: 11800
```

### skywalking-ui docker
```
FROM --platform=linux/amd64 apache/skywalking-ui:9.2.0

ENV SW_OAP_ADDRESS http://skywalking-oap:12800
ENV TZ Asia/Shanghai..
```

### Skywalking agent

1. 7.x版本中代理支持 JDK 8-14， 6.x版本支持JDK 1.6- JDK12 
2. 在SkyWalking发行包中找到agent文件夹
3. 配置config/agent.config中的agent.service_name。可以是任意的英文字符串。
4. 配置config/agent.config中的collector.backend_service。默认指向127.0.0.1:11800，表示仅作用于本地后端。可以修改配置为部署的skywalking OAP 集群地址
5. JVM参数中添加-javaagent:/path/to/skywalking-package/agent/skywalking-agent.jar，并且确保这个参数在-jar参数之前。
	`java  -javaagent:/Users/coder_wang/Workspace/skywalking-agent/skywalking-agent.jar -jar demo.jar`
6. 启动应用时设置 `SW_AGENT_NAME=trade-biz` 环境变量，这样在 skywalking 中就能对应显示你设置的应用名字，其它环境变量：
   ```
    export SW_AGENT_NAME=trade-biz # 配置 Agent 名字。一般来说，我们直接使用 Spring Boot 项目的 `spring.application.name` 。
	export SW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap:11800 # 配置 Collector 地址。
	export SW_AGENT_SPAN_LIMIT=2000 # 配置链路的最大 Span 数量。一般情况下，不需要配置，默认为 300 。主要考虑，有些新上 SkyWalking Agent 的项目，代码可能比较糟糕。
	export JAVA_AGENT=-javaagent:/home/skywalking-agent/skywalking-agent.jar # SkyWalking Agent jar 地址。
   ```

agent日志：
/Users/yunai/skywalking/apache-skywalking-apm-bin-es7/agent/agent/logs/skywalking-api.log

Java Agent 的默认配置在config/agent.config 中，常见的配置可参考[此处](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/#agent%E7%9A%84%E5%8F%AF%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8)

[日志集成](https://skywalking.apache.org/docs/skywalking-java/v8.12.0/en/setup/service-agent/java-agent/application-toolkit-logback-1.x/)：
```
<dependency>
	<groupId>org.apache.skywalking</groupId>
	<artifactId>apm-toolkit-logback-1.x</artifactId>
	<version>8.12.0</version>
</dependency>

<appender name="grpc-log" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
	<encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
		<layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
			<Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
		</layout>
	</encoder>
</appender>

<logger name="com.demo" level="INFO">
	<appender-ref ref="grpc-log" />
</logger>
```