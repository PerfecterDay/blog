# Consul 服务注册
{docsify-updated}

> https://developer.hashicorp.com/consul/docs/register

## 概览
对于服务发现，Consul 服务核心工作流程包括三个阶段：
1. 定义服务和健康检查： 服务定义可以定义服务的各个方面，包括如何被网络中的其他服务发现。也可以在服务定义中定义健康检查，以验证服务的健康状况。
2. 注册服务和健康检查： 定义服务和健康检查后，必须向 Consul 代理注册。
3. 查询服务： 注册服务和健康检查后，网络中的其他服务可使用 DNS 执行静态或动态查询，以访问您的服务。有关在数据中心发现服务的不同方法的更多信息，请参阅 DNS 使用概述。

## 定义服务


## 监听服务变化
`ConsulCatalogWatch` 可以监听 Consul 中服务的变化， 发布 `HeartbeatEvent` 事件。
