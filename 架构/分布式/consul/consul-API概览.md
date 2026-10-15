#  Consul API 概览
{docsify-updated}
> https://developer.hashicorp.com/consul  
> https://kingfree.gitbook.io/consul/getting-started/agent

- [Consul API 概览](#consul-api-概览)
    - [consul 命令行-CLI](#consul-命令行-cli)
    - [Consul HTTP API Overview](#consul-http-api-overview)
      - [连接你的服务](#连接你的服务)
      - [启用零信任的网络安全](#启用零信任的网络安全)
      - [观察你的网络](#观察你的网络)
      - [管理 Consul](#管理-consul)
      - [动态地配置](#动态地配置)
    - [HTTP API](#http-api)


### consul 命令行-CLI
```
consul
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    acl            Interact with Consul's ACLs
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    connect        Interact with Consul Connect
    debug          Records a debugging archive for operators
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    intention      Interact with Connect service intentions
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    login          Login to Consul using an auth method
    logout         Destroy a Consul token created with login
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    peering        Create and manage peering connections between Consul clusters
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    services       Interact with services
    snapshot       Saves, restores and inspects snapshots of Consul server state
    tls            Builtin helpers for creating CAs and certificates
    troubleshoot   Provides tools to troubleshoot Consul's service mesh configuration
    validate       Validate config files/directories
	version        Prints the Consul version
    watch          Watch for changes in Consul
```

### Consul HTTP API Overview 
Consul HTTP API是一个RESTful接口，允许你在网络中利用Consul功能。本主题提供了关于不同工作流的基本API端点的指导。参考[HTTP API结构](/consul/api-docs/api-structure)文档，了解如何与Consul HTTP API进行交互和认证。

#### 连接你的服务
使用以下API端点来配置和连接你的服务。
- [`/catalog`](https://developer.hashicorp.com/consul/api-docs/catalog)：注册和取消注册节点、服务和健康检查。
- [`/health`](https://developer.hashicorp.com/consul/api-docs/health)：在启用健康检查时查询节点的健康状况。
- [`/query`](https://developer.hashicorp.com/consul/api-docs/query)：在Consul中创建和管理预备查询。预备查询允许你注册一个复杂的服务查询，然后再发送。
- [`/coordinate`](https://developer.hashicorp.com/consul/api-docs/coordinate)：查询本地数据中心的节点以及本地数据中心和远程数据中心的Consul服务器的网络坐标。

以下端点是针对服务网状的：

- [`/config`]（https://developer.hashicorp.com/consul/api-docs/config）：创建、更新、删除和查询在Consul注册的中央配置条目。配置项定义了服务网中资源的默认行为。
- [`/agent/connect`]（https://developer.hashicorp.com/consul/api-docs/agent/connect）：与服务网中的本地代理进行交互。
- [`/connect`](https://developer.hashicorp.com/consul/api-docs/connect)：管理服务网的相关操作，包括服务意向（[`/connect/intentions`](https://developer.hashicorp.com/consul/api-docs/connect/intentions)）和服务网证书授权（CA）（[`/connect/ca`](https://developer.hashicorp.com/consul/api-docs/connect/ca)）。

#### 启用零信任的网络安全
以下API端点可以让你控制对网络中服务的访问和对Consul API的访问。
- [`/acl`]（https://developer.hashicorp.com/consul/api-docs/acl）：创建和管理令牌，以验证请求和授权访问网络中的资源。我们建议启用访问控制列表（ACL）以确保对Consul API、用户界面和CLI的访问。
- [`/connect/intentions`]（https://developer.hashicorp.com/consul/api-docs/connect/intentions）：创建和管理服务意向。

#### 观察你的网络
使用以下API端点可以实现网络的可观察性。
- [`/status`]（https://developer.hashicorp.com/consul/api-docs/status）：通过返回有关Consul服务器对等体的低级Raft信息来调试你的Consul数据中心。
- [`/agent/metrics`](https://developer.hashicorp.com/consul/api-docs/agent#view-metrics)：检索最近完成的间隔的度量。关于指标的更多信息，请参考 [Telemetry](/consul/docs/agent/telemetry)。

#### 管理 Consul
以下API端点可以帮助你管理Consul操作。
- [`/operator`](https://developer.hashicorp.com/consul/api-docs/operator)：执行集群级任务，如与Raft子系统交互或获取许可信息。
- [`/partition`](https://developer.hashicorp.com/consul/api-docs/admin-partitions)：在Consul中创建和管理管理区或管理员分区。管理分区是Consul命名空间的超集，用于隔离资源组以降低操作开销。
- [`/namespace`](https://developer.hashicorp.com/consul/api-docs/namespaces)：在Consul中创建和管理命名空间。命名空间隔离了资源组，降低了操作的开销。
- [`/snapshot`](https://developer.hashicorp.com/consul/api-docs/snapshot)：在灾难发生时保存和恢复Consul服务器状态。
- [`/txn`](https://developer.hashicorp.com/consul/api-docs/txn)：在一个事务中应用多个操作，如更新目录和检索多个KV条目。

#### 动态地配置
以下API端点使你能够动态配置你的服务。
- [`/event`]（https://developer.hashicorp.com/consul/api-docs/event）：启动一个自定义事件，你可以用它来构建脚本和自动程序。
- [`/kv`](https://developer.hashicorp.com/consul/api-docs/kv)：添加、删除和更新存储在Consul KV商店的元数据。
- [`/session`](https://developer.hashicorp.com/consul/api-docs/session)：在Consul中创建和管理[session]（https://developer.hashicorp.com/consul/docs/dynamic-app-config/sessions）。你可以使用会话来建立分布式和细粒度的锁，以确保节点正确写入Consul KV存储。


### [HTTP API](https://developer.hashicorp.com/consul/api-docs/catalog#list-nodes-for-service)
查找所有注册的服务信息： `curl http://localhost:8500/v1/catalog/services`
查找所有注册的数据中心信息： `curl http://localhost:8500/v1/catalog/datacenters`
根据服务名查找某个具体服务信息： `curl http://localhost:8500/v1/catalog/service/{serviceName}`

`curl http://localhost:8500/v1/agent/checks`
查看处于 critical 状态的服务：`curl http://localhost:8500/v1/health/state/critical`

```
curl --header "X-Consul-Namespace: *" http://127.0.0.1:8500/v1/health/node/my-node
curl http://127.0.0.1:8500/v1/health/service/my-service?ns=default
curl http://127.0.0.1:8500/v1/agent/services
```

