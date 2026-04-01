> https://thenewstack.io/why-do-you-need-istio-when-you-already-have-kubernetes/

如果你听说过服务网格，并尝试过Istio，你可能会有以下问题。

+ 为什么Istio要在Kubernetes上运行？
+ Kubernetes和服务网状结构在云原生应用架构中的作用分别是什么？
+ Istio扩展了Kubernetes的哪些方面？它能解决什么问题？
+ Kubernetes、Envoy和Istio之间的关系是什么？
本文将带你了解Kubernetes和Istio的内部运作情况。此外，我还将介绍Kubernetes中的负载均衡方法，并解释为什么有了Kubernetes就需要Istio。

Kubernetes本质上是通过声明性配置进行应用生命周期管理，而服务网状结构本质上是提供应用间流量、安全管理和可观察性。如果你已经使用Kubernetes建立了一个稳定的应用平台，你如何为服务之间的调用设置负载均衡和流量控制？这就是服务网状结构出现的地方。

Envoy引入了xDS协议，该协议得到了各种开源软件的支持，如Istio、MOSN等。Envoy将xDS贡献给服务网状结构或云原生基础设施。Envoy本质上是一个现代版的代理，可以通过API进行配置，在此基础上衍生出许多不同的使用场景--如API网关、服务网中的挎包代理和边缘代理。

这篇文章包含以下内容。

对kube-proxy的作用的描述。
Kubernetes对微服务管理的限制。
Istio服务网的功能介绍。
Kubernetes、Envoy和Istio服务网中一些概念的比较。

### Kubernetes与服务网格
下图显示了Kubernetes和service mesh中的服务访问关系（每个pod模型一个sidecar）。

<center>
<img src="pics/k8s-native.png" >
<img src="pics/service-mesh.png" >
</center>

#### 流量转发
Kubernetes集群中的每个节点都部署了一个kube-proxy组件，它与Kubernetes API服务器进行通信，获取集群中的服务信息，然后设置iptables规则，将服务请求直接发送到相应的Endpoint（属于同一服务组的pod）。

#### 服务发现
<center><img src="pics/service-disc.png" ></center>

Istio可以跟随Kubernetes中的服务注册，也可以通过控制平面中的平台适配器与其他服务发现系统对接；然后用数据平面的透明代理生成数据平面配置（使用CRD，存储在etcd中）。数据平面的透明代理被作为一个sidecar容器部署在每个应用服务的pod中，所有这些代理都需要请求控制平面来同步代理配置。代理是 "透明 "的，因为应用容器完全不知道代理的存在。过程中的kube-proxy组件也需要拦截流量，只是kube-proxy拦截往返于Kubernetes节点的流量--而sidecar代理拦截往返于pod的流量。

### 服务网的劣势
由于Kubernetes的每个节点上都有很多pod在运行，把原来的kube-proxy路由转发功能放在每个pod中会增加响应延迟--由于sidecar拦截流量时有更多的跳数--并且消耗更多的资源。为了以细粒度的方式管理流量，将添加一系列新的抽象概念。这将进一步增加用户的学习成本，但随着技术的普及，这种情况将慢慢得到缓解。


### Kube-Proxy的不足之处
首先，如果转发的pod不能正常服务，它不会自动尝试另一个pod。每个pod都有一个健康检查机制，当一个pod出现健康问题时，kubelet会重新启动pod，kube-proxy会删除相应的转发规则。另外，nodePort类型的服务不能添加TLS或更复杂的消息路由机制。

Kube-proxy实现了跨Kubernetes服务的多个pod实例的流量负载均衡，但如何对这些服务之间的流量进行细粒度控制--比如按百分比划分流量到不同的应用版本（都是同一服务的一部分，但在不同的部署中），或者做金丝雀发布（灰度发布）和蓝绿色发布？

Kubernetes社区给出了一种使用部署来做金丝雀发布的方法，这基本上是一种通过修改pod的标签将不同的pod分配给部署的服务的方法。


### Kubernetes Ingress vs. Istio Gateway
如上所述，kube-proxy只能在一个Kubernetes集群内路由流量。Kubernetes集群的pod位于由CNI创建的网络中。一个ingress--在Kubernetes中创建的资源对象--是为集群外的通信而创建的。它由位于Kubernetes边缘节点的ingress控制器驱动，负责管理南北交通。Ingress必须与各种Ingress控制器对接，如nginx ingress控制器和traefik。Ingress只适用于HTTP流量，使用起来很简单。它只能通过匹配有限的字段来路由流量--如服务、端口、HTTP路径等。这使得它无法路由TCP流量，如MySQL、Redis和各种RPCs。这就是为什么你看到人们在ingress资源注释中写nginx配置语言的原因。直接路由南北流量的唯一方法是使用服务的LoadBalancer或NodePort，前者需要云供应商支持，后者需要额外的端口管理。

Istio Gateway的功能与Kubernetes Ingress类似，它负责集群的南北向流量。Istio网关描述了一个负载平衡器，用于承载进出网状边缘的连接。该规范描述了一组开放的端口和这些端口使用的协议、用于负载平衡的SNI配置等。Gateway是一个CRD扩展，它也重用了sidecar代理的功能；详细配置见Istio网站。

### Envoy
Envoy是Istio中默认的边车代理。Istio基于Enovy的xDS协议来扩展其控制平面。在谈论Envoy的xDS协议之前，我们需要先熟悉Envoy的基本术语。下面是Envoy中的基本术语及其数据结构的列表，更多细节请参考Envoy文档。