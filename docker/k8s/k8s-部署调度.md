# K8S 调度、抢占和驱逐
{docsify-updated}

> https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity

在 Kubernetes 中，调度 (scheduling) 指的是确保 Pod 匹配到合适的节点， 以便 kubelet 能够运行它们。 抢占 (Preemption) 指的是终止低优先级的 Pod 以便高优先级的 Pod 可以调度运行的过程。 驱逐 (Eviction) 是在资源匮乏的节点上，主动让一个或多个 Pod 失效的过程。


### 将 Pod 指派到特定条件的节点
可以约束一个 Pod 以便限制其只能在特定的节点上运行， 或优先在特定的节点上运行。有几种方法可以实现这点，推荐的方法都是用 标签选择算符来进行选择。 通常这样的约束不是必须的，因为调度器将自动进行合理的放置（比如，将 Pod 分散到节点上， 而不是将 Pod 放置在可用资源不足的节点上等等）。但在某些情况下，你可能需要进一步控制 Pod 被部署到哪个节点。例如，确保 Pod 最终落在连接了 SSD 的机器上， 或者将来自两个不同的服务且有大量通信的 Pod 被放置在同一个可用区。