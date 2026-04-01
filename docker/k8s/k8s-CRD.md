# k8s CRD
{docsify-updated}

> https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#create-a-customresourcedefinition

## 定制资源
资源（Resource） 是 Kubernetes API 中的一个端点， 其中存储的是某个类别的 API 对象的一个集合。 例如内置的 Pod 资源包含一组 Pod 对象。

定制资源（Custom Resource） 是对 Kubernetes API 的扩展，不一定在默认的 Kubernetes 安装中就可用。定制资源所代表的是对特定 Kubernetes 安装的一种定制。 不过，很多 Kubernetes 核心功能现在都用定制资源来实现，这使得 Kubernetes 更加模块化。

定制资源可以通过动态注册的方式在运行中的集群内或出现或消失，集群管理员可以独立于集群更新定制资源。 一旦某定制资源被安装，用户可以使用 kubectl 来创建和访问其中的对象，就像他们为 Pod 这种内置资源所做的一样。

Kubernetes 提供了两种方式供你向集群中添加定制资源：
+ CRD(CustomResourceDefinitions) 相对简单，创建 CRD 可以不必编程。
+ API 聚合需要编程， 但支持对 API 行为进行更多的控制，例如数据如何存储以及在不同 API 版本间如何转换等。

### CustomResourceDefinitions
CustomResourceDefinition API 资源允许你定义定制资源。 定义 CRD 对象的操作会使用你所设定的名字和模式定义（Schema）创建一个新的定制资源， Kubernetes API 负责为你的定制资源提供存储和访问服务。 CRD 对象的名称必须是有效的 DNS 子域名， 该名称由定义的资源名称及其 API 组派生而来。此外，由 CRD 定义的某种对象/资源的名称也必须是有效的 DNS 子域名。

CRD 使得你不必编写自己的 API 服务器来处理定制资源，不过其背后实现的通用性也意味着你所获得的灵活性要比 API 服务器聚合少很多。

#### 创建 CustomResourceDefinition
当创建新的 CustomResourceDefinition（CRD）时，Kubernetes API 服务器会为你所指定的每个版本**生成一个新的 RESTful 资源路径**。 基于 CRD 对象所创建的自定义资源可以是名字空间作用域的，也可以是集群作用域的， 取决于 CRD 对象 `spec.scope` 字段的设置。

与其它的内置对象一样，删除名字空间也将删除该名字空间中的所有自定义对象。 `CustomResourceDefinitions` 本身是无名字空间的，可在所有名字空间中访问。

如果你将下面的 CustomResourceDefinition 保存到 `resourcedefinition.yaml` 文件：
```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API：/apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的驼峰命名（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

然后执行： `kubectl apply -f resourcedefinition.yaml`.   
这样一个新的受名字空间约束的 RESTful API 端点会被创建在： `/apis/stable.example.com/v1/namespaces/*/crontabs/...`

此端点 URL 自此可以用来创建和管理定制对象。对象的 kind 将是来自你上面创建时所用的规约中指定的 CronTab。
创建端点的操作可能需要几秒钟。你可以监测你的 CustomResourceDefinition 的 Established 状况变为 true，或者监测 API 服务器的发现信息等待你的资源出现在那里。

删除CRD ： `kubectl delete -f resourcedefinition.yaml`

#### 创建定制对象
在创建了 CustomResourceDefinition 对象之后，你可以创建**定制对象**（Custom Objects）。定制对象可以包含定制字段。这些字段可以包含任意的 JSON 数据。 在下面的例子中，在类别为 CronTab 的定制对象中，设置了 `cronSpec` 和 `image` 定制字段。类别 `CronTab` 来自你在上面所创建的 CRD 的规约。

将下面的 YAML 保存到 `my-crontab.yaml`：
```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

执行： `kubectl apply -f my-crontab.yaml`

如果使用 `curl http://127.0.0.1:8001/apis/stable.example.com/v1/namespaces/default/crontabs\?watch\=true` 监听了资源的变化，就能看到如下输出：
<center><img src="pics/k8s-watch.png"></center>

查看 crontab 资源： `kubectl get crontab` ，查看详细信息可以 `kubectl get ct -o yaml`

#### 定制控制器
就定制资源本身而言，它只能用来存取结构化的数据。 当你将**定制资源**与**定制控制器**（Custom Controller） 结合时， 定制资源就能够提供真正的声明式 API（Declarative API）。

Kubernetes 声明式 API 强制对职权做了一次分离操作。 你声明所用资源的期望状态，而 Kubernetes 控制器使 Kubernetes 对象的当前状态与你所声明的期望状态保持同步。 声明式 API 的这种机制与命令式 API（你指示服务器要做什么，服务器就去做什么）形成鲜明对比。

你可以在一个运行中的集群上部署和更新定制控制器，这类操作与集群的生命周期无关。 定制控制器可以用于任何类别的资源，不过它们与定制资源结合起来时最为有效。 Operator 模式就是将定制资源与定制控制器相结合的。 你可以使用定制控制器来将特定于某应用的领域知识组织起来，以编码的形式构造对 Kubernetes API 的扩展。

#### 工作原理
```
你 apply 一个 CRD.yaml --> kube-apiserver 注册该资源类型 --> etcd 记录 CRD 元信息
         ↓
你创建一个 CustomResource（CR）
         ↓
kube-apiserver 接收 POST /apis/mygroup/v1/myresources
         ↓
etcd 存储这个 CR 的具体内容（YAML 对象）
         ↓
（可选）你写的 Controller 监听这些 CR
            ↓
    Informer（缓存 + Watch 管理器）
            ↓
    Kube-apiserver 的 GET /api/v1/myresources?watch=true 接口（HTTP 长连接）
            ↓
    etcd（底层存储，触发事件）
            ↓
Controller 根据 CR 内容执行自定义逻辑，如部署服务、修改资源
```

#### 实战
https://github.com/kubernetes/sample-controller

```
# assumes you have a working kubeconfig, not required if operating in-cluster
go build -o sample-controller .
./sample-controller -kubeconfig=$HOME/.kube/config

# create a CustomResourceDefinition
kubectl create -f artifacts/examples/crd-status-subresource.yaml

# create a custom resource of type Foo
kubectl create -f artifacts/examples/example-foo.yaml

# check deployments created through the custom resource
kubectl get deployments
```

效果：控制器会监听K8S集群的变化，当在K8S集群中部署自定义的资源时，会部署一个 nginx pod。

1. 安装 kubebuilder  
```
# download kubebuilder and install locally.
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
```

2. 初始化项目
```
kubebuilder init --domain my.domain --repo my.domain/guestbook
```

3. 创建API
```
kubebuilder create api --group webapp --version v1 --kind Guestbook
make manifests

make install
make run
```


### API 服务器聚合 
通常，Kubernetes API 中的每个资源都需要处理 REST 请求和管理对象持久性存储的代码。 Kubernetes API 主服务器能够处理诸如 Pod 和 Service 这些内置资源， 也可以按通用的方式通过 CRD 来处理定制资源。

聚合层（Aggregation Layer） 使得你可以通过编写和部署你自己的 API 服务器来为定制资源提供特殊的实现。 主 API 服务器将针对你要处理的定制资源的请求全部委托给你自己的 API 服务器来处理， 同时将这些资源提供给其所有客户端。