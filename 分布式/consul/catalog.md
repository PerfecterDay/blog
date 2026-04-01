#  Catalog API

`/catalog`端点在Consul中注册和注销节点、服务和检查。Consul中节点和服务的注册与取消。`/catalog`不应该与`/agent`混淆，因为一些API方法看起来很相似。



## 注册实体

## 查询节点
```
GET /v1/catalog/nodes
```

## 查询服务列表
```
GET /v1/catalog/services
```

## 注销服务
```
curl -X PUT http://localhost:8500/v1/catalog/deregister -H "Content-Type: application/json" -d '{"Datacenter": "dc1", "Node": "consul-server-2", "ServiceID": "trade-center-gmt-5755564b9-4cqmv" }'
```