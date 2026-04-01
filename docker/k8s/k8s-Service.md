# k8s-Service
{docsify-updated}

我们不应该期望 Kubernetes Pod 是健壮的，而是要假设 Pod 中的容器很可能因为各种原因发生故障而死掉。Deployment 等 controller 会通过动态创建和销毁 Pod 来保证应用整体的健壮性。换句话说，Pod 是脆弱的，但应用是健壮的。  
每个 Pod 都有自己的 IP 地址。当 controller 用新 Pod 替代发生故障的 Pod 时，新 Pod 会分配到新的 IP 地址。这样就产生了一个问题：  
如果一组 Pod 对外提供服务（比如 HTTP），它们的 IP 很有可能发生变化，那么客户端如何找到并访问这个服务呢？

Kubernetes 给出的解决方案是 Service。


## Service 创建
Kubernetes Service 从逻辑上代表了一组 Pod，具体是哪些 Pod，则是由selector 来筛选带有特定label 值的Pod。**Service 有自己 IP，而且这个 IP 是不变的**。客户端只需要访问 Service 的 IP，Kubernetes 则负责建立和维护 Service 与 Pod 的映射关系。无论后端 Pod 如何变化，对客户端不会有任何影响，因为 Service 没有变。

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: '2023-02-08T06:22:48Z'
  labels:
    micrometer-prometheus-discovery: 'true'
  name: user-center-svc
  namespace: default
  resourceVersion: '92021626'
  uid: fef0d177-5e7e-4817-8f18-e8408b503410
spec:
  clusterIP: 192.168.162.123
  clusterIPs:
    - 192.168.162.123
  internalTrafficPolicy: Cluster
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
  ports:
    - name: user-center-svc-8913
      port: 8913
      protocol: TCP
      targetPort: 8913
    - name: user-center-svc-8091
      port: 8091 //service 端口
      protocol: TCP
      targetPort: 8091 //POD端口
  selector:
    app: user-center-service
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

## Service IP 原理
Service Cluster IP 是一个虚拟 IP，是由 Kubernetes 节点上的 iptables 规则管理的。  
iptables 将访问 Service 的流量转发到后端 Pod，而且使用类似轮询的负载均衡策略。Cluster 的每一个节点都配置了相同的 iptables 规则，这样就确保了整个 Cluster 都能够通过 Service 的 Cluster IP 访问 Service。

在 Cluster 中，除了可以通过 Cluster IP 访问 Service，Kubernetes 还提供了更为方便的 DNS 访问。
k8s 部署时会默认安装 kube-dns 组件。
kube-dns 是一个 DNS 服务器。每当有新的 Service 被创建，kube-dns 会添加该 Service 的 DNS 记录。Cluster 中的 Pod 可以通过 <SERVICE_NAME>.<NAMESPACE_NAME> 访问 Service。

通过 namespace: kube-public 指定资源所属的 namespace。多个资源可以在一个 YAML 文件中定义，用 --- 分割

## Service的种类
除了 Cluster 内部可以访问 Service，很多情况我们也希望应用的 Service 能够暴露给 Cluster 外部。Kubernetes 提供了多种类型的 Service，默认是 ClusterIP。

### ClusterIP
Service 通过 Cluster **内部的 IP**对外提供服务，只有 **Cluster 内的节点和 Pod 可访问**，这是默认的 Service 类型

#### Headlsess service
默认的 ClusterIP 类型的 service，会生成一个一个内部的IP，而 headless service 会返回后端pod 的IP。如下：
<center><img src="pics/headless-service.png" width="50%"></center>

headless service 的clusterIP 为 None：
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-app-headless
spec:
  clusterIP: None  # 👈 关键点，表示这是 headless service
  selector:
    app: my-app
  ports:
  - port: 8080 # 👈 必须写 Pod 实际监听的端口
    targetPort: 8080 
```

注意：headless service 不会经过 kube-proxy 做转发，所以也不会有端口映射。客户端通过服务名访问服务名时，解析到的是 pod 的IP，所以端口也必须是 pod 真实的端口，否则会连接不上。


### NodePort
Service 通过 Cluster 节点的静态端口**对外**提供服务。**Cluster 外部**可以通过 `<NodeIP>:<NodePort>` 访问 Service。

### LoadBalancer
Service 利用 cloud provider 特有的 load balancer 对外提供服务，cloud provider 负责将 load balancer 的流量导向 Service。目前支持的 cloud provider 有 GCP、AWS、Azur 等。
```
apiVersion: v1
kind: Service
metadata:
annotations:
	service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cert-id: 5101273941756630_186597c33ea_-1637077474_-1736767758
	service.beta.kubernetes.io/alibaba-cloud-loadbalancer-spec: slb.s1.small
	service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
creationTimestamp: '2023-02-21T02:39:56Z'
finalizers:
	- service.k8s.alibaba/resources
labels:
	service.beta.kubernetes.io/hash: 7ae73a1e15324eb203b3c2d08789d5f91152c4b56ca39e8c59b8990b
	service.k8s.alibaba/loadbalancer-id: lb-3nsiwzexohm9i5208ktn3
managedFields:
	- apiVersion: v1
	fieldsType: FieldsV1
	fieldsV1:
		'f:metadata':
		'f:finalizers':
			.: {}
			'v:"service.k8s.alibaba/resources"': {}
	manager: cloud-controller-manager
	operation: Update
	time: '2023-02-21T02:39:56Z'
	- apiVersion: v1
	fieldsType: FieldsV1
	fieldsV1:
		'f:metadata':
		'f:labels':
			.: {}
			'f:service.beta.kubernetes.io/hash': {}
			'f:service.k8s.alibaba/loadbalancer-id': {}
		'f:status':
		'f:loadBalancer':
			'f:ingress': {}
	manager: cloud-controller-manager
	operation: Update
	subresource: status
	time: '2023-02-21T02:40:05Z'
	- apiVersion: v1
	fieldsType: FieldsV1
	fieldsV1:
		'f:metadata':
		'f:annotations':
			.: {}
			'f:service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cert-id': {}
			'f:service.beta.kubernetes.io/alibaba-cloud-loadbalancer-spec': {}
			'f:service.beta.kubernetes.io/alicloud-loadbalancer-address-type': {}
		'f:spec':
		'f:allocateLoadBalancerNodePorts': {}
		'f:externalTrafficPolicy': {}
		'f:internalTrafficPolicy': {}
		'f:ipFamilyPolicy': {}
		'f:ports':
			.: {}
			'k:{"port":443,"protocol":"TCP"}':
			.: {}
			'f:name': {}
			'f:port': {}
			'f:protocol': {}
			'f:targetPort': {}
			'k:{"port":8000,"protocol":"TCP"}':
			.: {}
			'f:name': {}
			'f:port': {}
			'f:protocol': {}
			'f:targetPort': {}
		'f:selector': {}
		'f:sessionAffinity': {}
		'f:type': {}
	manager: ACK-Console Apache-HttpClient
	operation: Update
	time: '2024-01-04T02:45:53Z'
name: uc-idc-slb
namespace: default
resourceVersion: '157006083'
uid: 4f1781c9-7e7f-41a6-80f5-920f26d6ca69
spec:
allocateLoadBalancerNodePorts: true
clusterIP: 192.168.219.247
clusterIPs:
	- 192.168.219.247
externalTrafficPolicy: Local
healthCheckNodePort: 31615
internalTrafficPolicy: Cluster
ipFamilies:
	- IPv4
ipFamilyPolicy: SingleStack
ports:
	- name: uc-idc-slb-8000-10000
	nodePort: 31357
	port: 8000
	protocol: TCP
	targetPort: 10000
	- name: uc-idc-slb-443-10000
	nodePort: 32035
	port: 443
	protocol: TCP
	targetPort: 10000
selector:
	app: envoy-hk-idc
sessionAffinity: None
type: LoadBalancer
status:
loadBalancer:
	ingress:
	- ip: 10.4.152.180
```

### ExternalName  
**ExternalName类型的Service将集群外部的域名映射到集群内部的Service上，例如将外部的数据库域名映射到集群内部的Service名，那么就能在集群内部通过Service名直接访问**。  
该映射将集群的 DNS 服务器配置为返回具有该外部主机名值的 CNAME 记录。 无需创建任何类型代理。

## Envoy + service 导致的负载不均衡问题

#### 问题描述
线上服务使用 envoy 作为网关入口，envoy 中配置的上游集群是k8s service（默认的 clusterIp服务），envoy 和 后端服务同属同一个集群。然后发现后端服务的多个服务之间收到的请求负载严重失衡，如图所示（阿里云SLS中日志量在不同pod上的分布,一个pod 98%，另一个2%，说明流量几乎全部打到一个pod 上了）：
<center><img src="pics/k8s-service-1.png" width="50%"></center>
<center><img src="pics/k8s-service-2.png" width="50%"></center>

导致其中一个 pod 内存占用比另一个高很多。

#### 原因分析
https://medium.com/@lapwingcloud/dont-load-balance-grpc-or-http2-using-kubernetes-service-ae71be026d7f

1. Envoy 会通过 TCP 或 HTTP 请求，连接：
```
10.96.0.100:8000  # ClusterIP + 端口, 这个是 Kubernetes 为 Service 分配的虚拟 IP。
```

2. Kube-proxy（默认是 iptables 或 IPVS 模式）会对访问 ClusterIP 的流量进行转发：
```
10.96.0.100:8080 
  ↓（iptables 转发）
POD-A:10.244.1.12:8910
```
+ kube-proxy 会根据 Service 的 Endpoints 列表，做 随机轮询或 hash-based 负载均衡
+ 实际连接的目标是后端 Pod 的 IP 和端口（如 10.244.x.x:8910）

连接级负载均衡：Envoy 建立的每个连接，只会被 kube-proxy 转发**一次**，之后连接复用就不会变了（导致“粘连”问题）
负载策略由 kube-proxy 控制，Envoy 感知不到后端 Pod 的数量、健康等信息

```
[Envoy]  →  [ClusterIP:8080] (my-service)
            ↓ iptables/IPVS 路由
         [Pod-1:10.244.1.12:80]
         [Pod-2:10.244.2.15:80]
         [Pod-3:10.244.3.20:80]
```

#### 解决方案
使用 Headless service + strict_dns

DNS 会直接返回 Pod IP 列表，Envoy 可用 strict_dns 模式自己进行轮询与健康检查，控制权更强。

```
                        Kubernetes Service（ClusterIP）
                    ┌───────────────────────────────┐
                    │ my-service.default.svc.cluster.local
                    │ ClusterIP = 10.96.0.100:8080
                    └───────────────────────────────┘
                                  │
                                  ▼
                         Envoy 连接 10.96.0.100:8080
                                  │
                                  ▼
                          kube-proxy 转发连接
                     （iptables/IPVS 模式做负载均衡）
            ┌───────────────────┬─────────────────────┐
            ▼                   ▼                     ▼
       [Pod A]            [Pod B]              [Pod C]
    10.244.0.12:80     10.244.1.21:80       10.244.2.33:80

📌 关键点：
✔️ Envoy 发起连接到 ClusterIP  
❌ Envoy 看不到 Pod 的 IP  
❌ 不能自定义健康检查、权重、负载策略  
❌ 是连接级别负载均衡（不适合 gRPC/长连接）
```

```
                         Headless Service
                    ┌───────────────────────────────┐
                    │ my-service.default.svc.cluster.local
                    │ ClusterIP = None
                    └───────────────────────────────┘
                                  │
                       DNS 解析返回多个 A 记录
            ┌───────────────────┬─────────────────────┐
            ▼                   ▼                     ▼
       10.244.0.12        10.244.1.21           10.244.2.33

                   ↓ Envoy strict_dns 解析每个 Pod IP
                          ▼             ▼             ▼
                  [Pod A:8080]   [Pod B:8080]   [Pod C:8080]
                          ▲             ▲             ▲
                          └───── Envoy 做轮询、自定义 lb ─────┘

📌 关键点：
✔️ Envoy 自己解析 DNS，获得 Pod IP  
✔️ 支持负载策略（round_robin、least_request 等）  
✔️ 支持健康检查、熔断、连接控制  
✔️ 更适合高性能场景或服务网格
✔️ 但是使用这种服务有一个限制就是，k8s服务的端口必须与 deployment 的端口一致，因为是直接连接pod 的端口，没有 k8s 帮忙转发
```