# POD 与 优雅停机
{docsify-updated}

Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

Pod（就像在鲸鱼荚或者豌豆荚中）是一组（一个或多个） 容器； 这些容器共享存储、网络、以及怎样运行这些容器的规约。 Pod 中的内容总是并置（colocated）的并且一同调度，在共享的上下文中运行。 Pod 所建模的是特定于应用的 “逻辑主机”，其中包含一个或多个应用容器， 这些容器相对紧密地耦合在一起。 在非云环境中，在相同的物理机或虚拟机上运行的应用类似于在同一逻辑主机上运行的云应用。

Pod 的共享上下文包括一组 Linux 名字空间、控制组（cgroup）和可能一些其他的隔离方面， 即用来隔离容器的技术。 在 Pod 的上下文中，每个独立的应用可能会进一步实施隔离。

Pod 类似于共享名字空间并共享文件系统卷的一组容器。

Kubernetes 集群中的 Pod 主要有两种用法：
+ 运行单个容器的 Pod。"每个 Pod 一个容器"模型是最常见的 Kubernetes 用例； 在这种情况下，可以将 Pod 看作单个容器的包装器，并且 Kubernetes 直接管理 Pod，而不是容器。
+ 运行多个协同工作的容器的 Pod。 Pod 可以封装由紧密耦合且需要共享资源的多个并置容器组成的应用。 这些位于同一位置的容器构成一个内聚单元。

## POD阶段
Pending → Running → Succeeded / Failed

+ `Pending`: Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
+ `Running`: Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
+ `Succeeded`: Pod 中的所有容器都已成功结束，并且不会再重启。
+ `Failed`: Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止，且未被设置为自动重启。
+ `Unknown`: 因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

## 优雅关闭问题
+ https://www.baeldung.com/linux/exec-command-in-shell-script
+ https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination
+ https://zhuanlan.zhihu.com/p/356125462
+ https://www.51cto.com/article/699592.html

问题： 阿里云生产环境有个服务，使用 consul 注册服务。但是 consul 里边的注册信息没有 deregister 掉，而且健康检查也是成功。导致调用方调用失败。经过分析，主要有两点很奇怪：
1. 为什么更新的时候没有 deregister 掉注册服务信息？
2. 为什么pod 已经不存在了，健康检查还能成功？

### 为什么更新的时候没有 deregister 掉注册服务信息？
consul 会在优雅停机时自动 `deregister` 掉注册的服务信息：
```
2025-10-22 15:54:36.071 [SpringApplicationShutdownHook] INFO  o.s.c.c.s.ConsulServiceRegistry   - Deregistering service with consul: user-center-service-8050
```
没有 deregister 掉，说明没有触发优雅停机。

由于 Pod 所代表的是在集群中节点上运行的进程，当不再需要这些进程时允许其优雅关闭是很重要的。 一般不应武断地使用 `KILL` 信号终止它们，导致这些进程没有机会完成清理操作。

K8S 优雅停机设计的目标是令你能够请求删除进程，并且知道进程何时被终止，同时也能够确保删除操作终将完成。 当你请求删除某个 Pod 时，集群会记录并跟踪 Pod 的优雅终止周期， 而不是直接强制地杀死 Pod。在存在强制关闭设施的前提下， kubelet 会尝试优雅地终止 Pod。

通常 Pod 优雅终止的过程为：**kubelet 先发送一个带有优雅超时限期的 `SIGTERM` 信号到每个容器中的主进程（PID为1的进程），将请求发送到容器运行时来尝试停止 Pod 中的容器。 停止容器的这些请求由容器运行时以异步方式处理。 这些请求的处理顺序无法被保证。许多容器运行时遵循容器镜像内定义的 `STOPSIGNAL` 值， 如果不同，则发送容器镜像中配置的 `STOPSIGNAL` ，而不是 `SIGTERM` 信号。**

在 Linux 上有了容器的概念之后，一旦容器建立了自己的 Pid Namespace(进程命名空间)，这个 Namespace 里的进程号也是从 1 开始标记的。所以，容器的 init 进程也被称为 1 号进程。你只需要记住：1 号进程是第一个用户态的进程，由它直接或者间接创建了 Namespace 中的其他进程。 

每个Docker容器都是一个PID命名空间，这意味着容器中的进程与主机上的其他进程是隔离的。PID命名空间是一棵树，从PID 1开始，通常称为init。  

注意：当你运行一个Docker容器时，镜像的 `ENTRYPOINT` 就是你的根进程，即PID 1(如果你没有 `ENTRYPOINT` ，那么 `CMD` 就会作为根进程，你可能配置了一个shell脚本，或其他的可执行程序，容器的根进程具体是什么，完全取决于你的配置)。

**一旦超出了优雅终止限期，容器运行时会向所有剩余进程发送 `SIGKILL` 信号，之后 Pod 就会被从 API 服务器上移除。 如果 kubelet 或者容器运行时的管理服务在等待进程终止期间被重启， 集群会从头开始重试，赋予 Pod 完整的优雅终止限期。**

回到我们的问题，我们的镜像中是通过 `start.sh` 脚本来启动 java 程序的：
```
#!/usr/bin/env bash
java -jar  -Dspring.profiles.active=uat app.jar
```
因此，POD 启动后，这个 `start.sh` 的shell 进程是PID为1 的进程，而 java 进程则是它的子进程。根据上边的文档， shell 并不会将 `SIGTERM` 信号传递给子进程，因此 java 进程实际上并没有机会执行 `deregister` 操作。

### 为什么pod 已经不存在了，健康检查还能成功？
即使 java 进程没有主动 deregister 操作，但是因为有 consul 的健康检查，理论上是会标记服务为 critical 的，并最终 deregister 服务的。原因是集群中有多个 java 服务，每个服务都是用了 actuator 作为健康检查的 endpoint，并且使用了相同的端口，这就导致了各个服务的健康检查 URL 都是一样的。

所以，如果执行 K8S 更新操作时，如果某个 POD 调度后使用了与注册服务 POD 之前的 IP，那么 Consul 就依然能够通过健康检查，并错误的认为服务已然存活，这样就会导致调用方调用失败


### 解决方案
1. 更改镜像中的启动脚本使java 服务能接受 `SIGTERM` 信号
```
#!/usr/bin/env bash
exec java -jar  -Dspring.profiles.active=uat app.jar
```

2. 设置 K8S 的优雅关闭
```
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30

lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]  # 模拟等待清理资源
```
