#  Agent API
{docsify-updated}

`/agent`端点用于与**本地Consul代理**进行交互。通常，服务和检查是在代理处注册的，然后由代理承担起保持该数据与集群同步的责任。例如，代理向目录注册服务和检查，并执行反熵以从中断中恢复。

## 概览
1. 检索主机信息
```
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/host
```
该端点返回代理正在运行的主机的信息，如CPU、内存和磁盘。

2. 列出成员
```
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/members
```
这个端点返回代理在集群gossip中看到的成员。由于八卦的性质，这最终是一致的：结果可能因代理而异。节点的强一致性视图则由/v1/catalog/nodes提供。  
这个端点还可以指定查询参数：
+ wan (bool: false)- 指定列出WAN成员，而不是LAN成员（这是默认的）。这只适用于以服务器模式运行的代理。
+ segment（string：""）- 指定要列出成员的网段。如果留空，当连接到服务器时，这将查询默认网段，当连接到客户端时，查询代理自己的网段（客户端只能是一个网段的一部分）。当查询服务器时，将此设置为特殊字符串_all将显示所有网段的成员。

3. 阅读配置
```
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/self
```
这个端点返回本地代理的配置和成员信息。Config元素包含配置的一个子集，其格式在不同版本之间不会有向后不兼容的变化。DebugConfig包含完整的运行时配置，但其格式会在没有通知或废弃的情况下改变。

4. 重新加载代理(Reload Agent)
```
curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/reload
```
这个端点指示代理重新加载其配置。在此过程中遇到的任何错误都会被返回。并非所有的配置选项都是可重新加载的，[支持选项的详情](https://developer.hashicorp.com/consul/docs/agent/config#reloadable-configuration)。

5. 查看指标
这个端点将转储最近完成的时间间隔的指标。
```
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/metrics
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/metrics?format=prometheus
```

6. 查看日志
这个端点从本地代理处流转日志，直到连接关闭。
```
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/monitor
curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/monitor?loglevel=info&logjson=true
```

7. 优雅地离开和关闭
这个端点会触发代理的优雅离开和关闭。它被用来确保其他节点看到代理是 "离开 "而不是 "失败"。离开的节点将不会试图在重新启动时用快照重新加入集群。  
对于服务器模式的节点，该节点会以一种优雅的方式从Raft对等体集合中移除。这一点至关重要，因为在某些情况下，非优雅的离开会影响集群的可用性。
```
curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/leave
```

## Checks 相关
1. 列出注册的检查：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/checks`
2. 注册检查：`curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/check/register`
3. 注销检查：`curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/check/deregister/{check_id}` check_id是路径参数
4. TTL报送健康信息：`curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1//agent/check/pass/{check_id}`

## Service 相关
1. 列出注册的服务：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/services`
2. 查看服务配置：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/service/{service_id}` service_id是路径参数
3. 根据**服务名**查询本地服务的健康状况：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/health/service/name/{service_name}`
	示例：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/health/service/name/trade-center-service`
4. 根据**服务ID**查询本地服务的健康状况：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500//v1/agent/health/service/id/{service_id}`
	示例：`curl -XGET http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/health/service/id/trade-center-gmt-68c7df96b7-wt6sn`  
5. 注册服务：`curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/service/register`
	示例：
	```
	curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/service/register \
	--data '{
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
	}'
	```
6. 取消服务注册： `curl -XPUT http://consul-cluster-server.consul.svc.cluster.local:8500/v1/agent/service/deregister/{service_id}`


## 注意事项
请注意因为是与本地 agent 交互的 API，有些查询的API并不能保证一致性（本地agent与 consul 集群信息不一致）:
<center>
	<img src="pics/agent-0.png" alt="">
	<img src="pics/agent-1.png" alt="">
</center>

所以如果是服务注册信息查询相关的功能不能使用 agent API，应使用 catalog/health API 来使用服务发现。