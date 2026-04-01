# k8s 概念及架构
{docsify-updated}

## 重要概念
1. **Cluster**  
	Cluster 是计算、存储和网络资源的集合，Kubernetes 利用这些资源运行各种基于容器的应用。
2. **Master**  
	Master 是Cluster 的大脑，它的主要职责是调度，即决定将应用放在哪里运行。Master 运行Linux 操作系统，可以是物理机或者虚拟机。为了实现高可用，可以运行多个Master。
3. **Node**  
	Node 的职责是运行容器应用。Node 由 Master 管理，Node 负责监控并汇报容器的状态， 同时根据Master 的要求管理容器的生命周期。Node 运行在Linux操作系统上，可以是物理机或者是虛拟机。Master同时也是一个Node
4. **Pod**  
	Pod 是 Kubernetes 的最小工作单元。每个 Pod **包含一个或多个容器**。Pod 中的容器会作为一个整体被 Master 调度到一个 Node 上运行。
5. **Controller**  
	K8S 通过 Controller 来管理 Pod。Controller 中定义了 Pod 的部署特性，比如有几个副本、在什么样的 Node 上运行等。K8S 提供了多种 Controller：
	+ Deployment：可以管理 Pod 的多个副本并确保 Pod 按照期望的状态运行
	+ ReplicaSet：实现了 Pod 的多副本管理。使用 Deployment 会自动创建 ReplicaSet，Deployment 通过 ReplicaSet 来管理 Pod 的多个副本，一般不直接使用 ReplicaSet。
	+ DaemonSet：用于每隔 Node 最多只运行一个 Pod 副本的场景。DaemonSet 通常用于运行 daemon。
	+ StatefulSet：能够保证 Pod 的每个副本在整个生命周期中是不变的，而其他 Controller 不提供这个功能。当某个 Pod 发生故障需要删除并重新启动时，Pod 的名称会发生变化，同时 StatefulSet 会保证副本按照固定顺序启动、更新或者删除。
	+ Job：用于运行结束就删除的应用，而其他 Controller 中的 Pod 通常是长期持续运行。
6. **Service**  
	Deployment 可以部署多个副本，每个Pod都有自己的IP，外界如何访问这些Pod的服务呢？通过IP吗？Pod会被频繁的重启和销毁重建，他们的IP会发生变化，用IP不能动态适应这种情形。  
	K8S Service 定义了外界访问一组特定 Pod 的方式。Service 有自己的 IP 和端口，Service 为 Pod 提供了负载均衡。  
	K8S 运行容器（Pod）与访问容器（Pod）这两个任分别由 Controller 和 Service 执行。
7. **Namespace**  
	可以将物理的 Cluster 逻辑上划分成多个虚拟 Cluster，每个 Cluster 就是一个 Namespace、不同的 Namespace 里的资源是完全隔离的。K8S 默认创建了两个 Namespace。
	+ default：创建资源时如果不指定，将被放到这个 Namespace 中
	+ kube-system：K8S 自己创建的系统资源将放到这个 Namespace 中


## k8s 架构
<center><img src="pics/k8s-components.jpg" alt="" width="60%"></center>

k8s 集群由 Masetr 和 Node 组成，节点上运行着若干 K8s 服务. Master 同时也是一个Node。

### Master 节点（控制平面）
Master 是 K8s 的大脑，运行着的Daemon服务包括 kube-apiserver、kube-scheduler、kube-controller-manager、etcd 和 pod 网络。为了保证高可用，可以同时有多个master节点。

1. Api Server(kube-apiserver)  
	Api Server 提供 HTTP/HTTPS RESTFUL API，即 K8S API。API server 是 K8s 集群的前端接口，各种客户端工具（CLI/UI）以及K8s其他组件可以通过它管理集群的各种资源。
2. Scheduler(kube-scheduler)  
	Scheduler 负责将Pod放在哪个Node上运行。Scheduler 在调度时会充分考虑到Cluster的拓扑结构、当前各个节点的负载以及对高可用、性能、数据亲和性的需求。
3. Controller Manager(kube-controller-manager)  
	Controller Manager 负责管理集群各种资源，保证资源处于预期状态。Controller Manager由多种controller 组成，包括 replication controller、endpoints controller、namespace controller、serviceaccount controller 等。
	不同的controller管理不同的资源。如，replication controller 管理 Deployment、Statefulset、DaemonSet 的生命周期，namespace controller管理Namespace资源。
4. etcd  
	etcd 负责保存 K8s的配置信息和各种资源的状态信息，当数据发生变化时，会快速地通知K8s相关组件。
5. Pod 网络  
	Pod要能够相互通信，K8s 集群必须部署 Pod 网络，flannel 是其中一个可选方案。

### Node 节点（数据平面）
Node 是Pod 运行的地方，K8S 支持 Docker、rkt等容器 runtime。Node 上运行的 K8s 组件有 kubelet 、 kube-proxy 和 Pod 网络。
1. kubelet  
	kubelet 是节点上运行的代理。当 Scheduler确定在某个Node上运行 Pod 后，会将Pod的具体配置信息发送给该节点 kubelet，kubelet 根据这些信息创建和运行容器，并向Master报告运行状态。
	kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。
2. kube-proxy  
	kube-proxy 是集群中每个节点上运行的网络代理,实现 Kubernetes Service 概念的一部分。service 在逻辑上代表了后端的多个 Pod ，外界通过 service 访问 Pod。service 接收到的请求是如何转发到 Pod 的呢 ？这就是 kube-proxy 的工作。
	每个 Node 上都会运行 kube-proxy 服务，kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。 它负责将访问service 的 TCP/UDP 数据流转发到后端的容器，并实现负载均衡。 
3. 容器运行环境(Container Runtime)  
	容器运行环境是负责运行容器的软件。  Kubernetes 支持多个容器运行环境: Docker、 containerd、cri-o、 rktlet 以及任何实现 Kubernetes CRI (容器运行环境接口)。
4. Pod 网络


## K8S部署示意图
<center><img src="pics/k8s-demo.jpg" alt="" width="60%"></center>

## K8S 一般原理
1. 执行 `kubectl apply -f demo.yaml` ，请求发给 `kube-apiserver`
2. `kube-apiserver` 校验 YAML、写入 `etcd`
3. `deployment controller` 发现新增 Deployment，创建 ReplicaSet
4. `replicaset controller` 发现需要 2 个 Pod，创建 2 个 Pod
5. `kube-scheduler` 监听到未绑定节点的 Pod，选择节点进行调度
6. Pod 被分配到某个 Node，记录在 `etcd` 中
7. 目标节点的 `kubelet` 监听到要运行的 Pod
8. `kubelet` 使用 `containerd` 拉取镜像并启动容器
9. Pod 启动后， `kubelet` 持续将运行状态汇报给 `kube-apiserver`
10. Controller 持续检查资源（如副本数）并且与预期的状态进行对比，，如果一致则认为“状态达成”，否则自动修复状态到预期状态 ➜ 比如某个 Pod 掉了，会自动重建


## Minikube安装运行k8s
1. 安装 Kubectl: `brew install kubectl`
2. 安装 Minikube: `brew install minikube`
3. 执行下列语句以使用本地docker镜像：`eval $(minikube docker-env)`
4. 启动 Minikube 并创建一个集群：`minikube start --driver=docker`
5. 查看集群信息： `kubectl cluster-info`、`kubectl get nodes` 。
6. 使用名为 echoserver 的镜像创建一个 Kubernetes Deployment: `kubectl create deployment hello-nginx --image=nginx`,删除的话使用：`kubectl delete deployment hello-nginx`
7. 查看集群的 deployments信息：`kubectl get deployments`
8. 查看集群的pods及日志信息： `kubectl get pods`,`kubectl describe pods [pod名]`，`kubectl log [pod名字]`
9. 将其作为 Service 公开: `kubectl expose deployment hello-nginx --type=NodePort --port=8080`
10. 获取暴露 Service 的 URL 以查看 Service 的详细信息: `kubectl describe services hello-nginx`
