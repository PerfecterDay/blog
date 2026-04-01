# gateway API
{docsify-updated}

Gateway API 具有三种稳定的 API 类别：
+ `GatewayClass` ： 定义一组具有配置相同的网关，由实现该类的控制器管理。
+ `Gateway` ： 定义流量处理基础设施（例如云负载均衡器）的一个实例。
+ `HTTPRoute` ： 定义特定于 HTTP 的规则，用于将流量从网关监听器映射到后端网络端点的表示。 这些端点通常表示为 Service。

Gateway API 被组织成不同的 API 类别，这些 API 类别具有相互依赖的关系，以支持组织中角色导向的特点。 一个 Gateway 对象只能与一个 GatewayClass 相关联；GatewayClass 描述负责管理此类 Gateway 的网关控制器。 各个（可以是多个）路由类别（例如 HTTPRoute）可以关联到此 Gateway 对象。 Gateway 可以对能够挂接到其 listeners 的路由进行过滤，从而与路由形成双向信任模型。

需要申明的一点是， Gateway API 规范只是一个 CRD 定义， 只要实现一个 controller 就能支持 CRD 的操作。有许多实现：
+ NGINX Gateway Fabric
+ Envoy Gateway
+ Kuma 
+ Traefik Proxy 
+ Gloo Gateway

## GatewayClass
`Gateway` 可以由不同的控制器实现，通常具有不同的配置。 `Gateway` 必须引用某 `GatewayClass` ，而后者中包含实现该类的控制器的名称。
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```
在此示例中，一个实现了 Gateway API 的控制器被配置为管理某些 GatewayClass 对象， 这些对象的控制器名为 example.com/gateway-controller。 归属于此类的 Gateway 对象将由此实现的控制器来管理。

## Gateway
Gateway 用来描述流量处理基础设施的一个实例。Gateway 定义了一个网络端点，该端点可用于处理流量， 即对 Service 等后端进行过滤、平衡、拆分等。 例如，Gateway 可以代表某个云负载均衡器，或配置为接受 HTTP 流量的集群内代理服务器。

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

在此示例中，流量处理基础设施的实例被编程为监听 80 端口上的 HTTP 流量。 由于未指定 addresses 字段，因此对应实现的控制器负责将地址或主机名设置到 Gateway 之上。 该地址用作网络端点，用于处理路由中定义的后端网络端点的流量。

## HTTPRoute
HTTPRoute 类别指定从 Gateway 监听器到后端网络端点的 HTTP 请求的路由行为。 对于服务后端，实现可以将后端网络端点表示为服务 IP 或服务的支持 EndpointSlices。 HTTPRoute 表示将被应用到下层 Gateway 实现的配置。 例如，定义新的 HTTPRoute 可能会导致在云负载均衡器或集群内代理服务器中配置额外的流量路由。

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```

在此示例中，来自 Gateway example-gateway 的 HTTP 流量， 如果 Host 的标头设置为 www.example.com 且请求路径指定为 /login， 将被路由到 Service example-svc 的 8080 端口。

## 请求数据流 
以下是使用 Gateway 和 HTTPRoute 将 HTTP 流量路由到服务的简单示例：
<center><img src="pics/gateway-request-flow.svg" alt=""></center>

在此示例中，实现为反向代理的 Gateway 的请求数据流如下：

1. 客户端开始准备 URL 为 http://www.example.com 的 HTTP 请求
2. 客户端的 DNS 解析器查询目标名称并了解与 Gateway 关联的一个或多个 IP 地址的映射。
3. 客户端向 Gateway IP 地址发送请求；反向代理接收 HTTP 请求并使用 Host: 标头来匹配基于 Gateway 和附加的 HTTPRoute 所获得的配置。
4. 可选的，反向代理可以根据 HTTPRoute 的匹配规则进行请求头和（或）路径匹配。
5. 可选地，反向代理可以修改请求；例如，根据 HTTPRoute 的过滤规则添加或删除标头。
6. 最后，反向代理将请求转发到一个或多个后端。


## GRPCRoute

## Policy

## ReferenceGrant