# K8S-对象
{docsify-updated}
> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/

- [K8S-对象](#k8s-对象)
  - [理解 Kubernetes 对象](#理解-kubernetes-对象)
    - [对象规约-Spec与状态-Status](#对象规约-spec与状态-status)
    - [对象描述](#对象描述)
      - [必需字段](#必需字段)
  - [对象名称和 ID](#对象名称和-id)
  - [标签和选择算符](#标签和选择算符)
    - [标签](#标签)
    - [标签选择算符](#标签选择算符)
  - [命名空间](#命名空间)
  - [注解](#注解)
  - [字段选择器](#字段选择器)


### 理解 Kubernetes 对象
在 Kubernetes 系统中，Kubernetes 对象是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。 具体而言，它们描述了如下信息：
+ 哪些容器化应用正在运行（以及在哪些节点上运行）
+ 可以被应用使用的资源
+ 关于应用运行时行为的策略，比如重启策略、升级策略以及容错策略

Kubernetes 对象是一种“意向表达（Record of Intent）”。一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。通过创建对象，你本质上是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的**期望状态**（Desired State）。

#### 对象规约-Spec与状态-Status 
几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： **对象 spec（规约）** 和 **对象 status（状态）**。 对于具有 spec 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： **期望状态**（Desired State）。

status 描述了对象的**当前状态**（Current State），它是由 Kubernetes 系统和组件设置并更新的。在任何时刻，Kubernetes 控制平面都一直在积极地管理着对象的实际状态，以使之达成期望状态。

#### 对象描述
创建 Kubernetes 对象时，必须提供对象的 spec，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（直接创建或经由 kubectl 创建）， API 请求必须在请求主体中包含 JSON 格式的信息。 大多数情况下，你需要提供 .yaml 文件为 kubectl 提供这些信息。kubectl 在发起 API 请求时，将这些信息转换成 JSON 格式。

##### 必需字段 
在想要创建的 Kubernetes 对象所对应的 .yaml 文件中，需要配置的字段如下：
+ `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
+ `kind` - 想要创建的对象的类别
+ `metadata` - 帮助唯一标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace
+ `spec` - 你所期望的该对象的状态，对每个 Kubernetes 对象而言，其 spec 的精确格式都是不同的

`kubectl --validate` 可以验证


### 对象名称和 ID
集群中的每一个对象都有一个名称来标识在同类资源中的唯一性。  
每个 Kubernetes 对象也有一个 UID 来标识在整个集群中的唯一性。

名称在同一资源的所有 API 版本中必须是唯一的。 这些 API 资源通过各自的 API 组、资源类型、命名空间（对于划分命名空间的资源）和名称来区分。 换言之，API 版本在此上下文中是不相关的。

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 .、..，也不可以包含 / 或 % 这些字符。

### 标签和选择算符

#### 标签
标签（Labels） 是附加到 Kubernetes 对象（比如 Pod）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 

**每个对象都可以定义一组键/值标签,每个键对于给定对象必须是唯一的。**
```
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

#### 标签选择算符 
与名称和 UID 不同， 标签不支持唯一性。通常，许多对象可以携带相同的标签。

通过标签选择算符，客户端/用户可以识别一组对象。标签选择算符是 Kubernetes 中的核心分组原语。API 目前支持两种类型的选择算符：**基于等值的和基于集合的**。 标签选择算符可以由逗号分隔的多个需求组成。 在多个需求的情况下，必须满足所有要求，因此逗号分隔符充当逻辑与（&&）运算符。

1. 基于等值或基于不等值的需求允许按标签键和值进行过滤  
   匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 可接受的运算符有 =、== 和 != 三种。 前两个表示相等（并且是同义词），而后者表示不相等。例如：
   ```
   environment = production
   tier != frontend
   ```

2. 基于集合的需求  
   基于集合的标签需求允许你通过一组值来过滤键。 支持三种操作符：in、notin 和 exists（只可以用在键标识符上）。例如：
   ```
   environment in (production, qa)
   tier notin (frontend, backend)
   partition
   !partition
   ```

一个 Service 指向的一组 Pod 是由标签选择算符定义的。同样，一个 ReplicationController 应该管理的 Pod 的数量也是由标签选择算符定义的。
```
selector:
  component: redis


selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
```

### 命名空间
在 Kubernetes 中，命名空间（Namespace） 提供一种机制，将同一集群中的资源划分为相互隔离的组。 同一命名空间内的资源名称要唯一，但跨命名空间时没有这个要求。 命名空间作用域仅针对带有命名空间的对象， （例如 Deployment、Service 等），这种作用域对集群范围的对象 （例如 StorageClass、Node、PersistentVolume 等）不适用。

Kubernetes 启动时会创建四个初始命名空间：
1. `default`
	Kubernetes 包含这个命名空间，以便于你无需创建新的命名空间即可开始使用新集群。
2. `kube-node-lease`
	该命名空间包含用于与各个节点关联的 Lease（租约）对象。 节点租约允许 kubelet 发送心跳， 由此控制面能够检测到节点故障。
3. `kube-public`
	所有的客户端（包括未经身份验证的客户端）都可以读取该命名空间。 该命名空间主要预留为集群使用，以便某些资源需要在整个集群中可见可读。 该命名空间的公共属性只是一种约定而非要求。
4. `kube-system`
	该命名空间用于 Kubernetes 系统创建的对象

#### 命名空间和DNS
当你创建一个服务时， Kubernetes 会创建一个相应的 DNS 条目。

该条目的形式是 `<服务名称>.<名字空间名称>.svc.cluster.local`，这意味着如果容器只使用 <服务名称>，它将被解析到本地名字空间的服务。这对于跨多个名字空间（如开发、测试和生产） 使用相同的配置非常有用。如果你希望跨名字空间访问，则需要使用完全限定域名（FQDN）。

### 注解
你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。 客户端程序（例如工具和库）能够获取这些元数据信息。

你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。**相反，注解不用于标识和选择对象**。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

注解和标签一样，是键/值对：
```
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### 字段选择器
“字段选择器(Field selectors)”允许你根据一个或多个资源字段的值筛选 Kubernetes 对象。 下面是一些使用字段选择器查询的例子：

```
metadata.name=my-service
metadata.namespace!=default
status.phase=Pending

kubectl get pods --field-selector status.phase=Running
```
可在字段选择器中使用 =、== 和 != （= 和 == 的意义是相同的）操作符。
