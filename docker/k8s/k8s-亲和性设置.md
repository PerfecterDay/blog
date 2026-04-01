# 亲和性
{docsify-updated}

> https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity

亲和性功能由两种类型的亲和性组成：

+ 节点亲和性功能类似于 nodeSelector 字段，但它的表达能力更强，并且允许你指定软规则。
+ Pod 间亲和性/反亲和性允许你根据其他 Pod 的标签来约束 Pod。

## 节点亲和性 
节点亲和性概念上类似于 nodeSelector， 它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点上。 节点亲和性有两种：

+ `requiredDuringSchedulingIgnoredDuringExecution` ： 调度器只有在规则被满足的时候才能执行调度。此功能类似于 `nodeSelector` ， 但其语法表达能力更强。
+ `preferredDuringSchedulingIgnoredDuringExecution` ： 调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。(软设置)

可以使用 Pod 规约中的 .spec.affinity.nodeAffinity 字段来设置节点亲和性。 例如，考虑下面的 Pod 规约：
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - cn-hongkong-b
```

## Pod 间亲和性与反亲和性
Pod 间亲和性与反亲和性使你可以基于已经在节点上运行的 Pod 的标签来约束 Pod 可以调度到的节点，而不是基于节点上的标签。

**Pod 间亲和性与反亲和性的规则格式为“如果 X 上已经运行了一个或多个满足规则 Y 的 Pod， 则这个 Pod 应该（或者在反亲和性的情况下不应该）运行在 X 上”。 这里的 X 可以是节点、机架、云提供商可用区或地理区域或类似的拓扑域， Y 则是 Kubernetes 尝试满足的规则。**

你通过标签选择算符的形式来表达规则（Y），并可根据需要指定选关联的名字空间列表。 Pod 在 Kubernetes 中是名字空间作用域的对象，因此 Pod 的标签也隐式地具有名字空间属性。 针对 Pod 标签的所有标签选择算符都要指定名字空间，Kubernetes 会在指定的名字空间内寻找标签。

你会通过 topologyKey 来表达拓扑域（X）的概念，其取值是系统用来标示域的节点标签键。 相关示例可参见常用标签、注解和污点。

示例(spec.template.spec 下)：
```
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - wealth-h5
        topologyKey: topology.kubernetes.io/zone
```
上述设置表示如果 zone 里已经有标签 `app: wealth-h5` 的 pod 存在，则不应该在这里部署。这样可以将pod 副本部署到多个不同的 zone 增加可用性/容灾。