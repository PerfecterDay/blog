# K8S 集群创建与交互
{docsify-updated}
- [K8S 集群创建与交互](#k8s-集群创建与交互)
    - [使用 podman-desktop 创建 k8s 集群](#使用-podman-desktop-创建-k8s-集群)
    - [使用 kubectl 与集群交互(https://kubernetes.io/zh-cn/docs/reference/kubectl/cheatsheet/)](#使用-kubectl-与集群交互httpskubernetesiozh-cndocsreferencekubectlcheatsheet)
      - [Kubectl 上下文和配置](#kubectl-上下文和配置)
      - [创建对象](#创建对象)
      - [查看和查找资源](#查看和查找资源)
      - [版本更新与回退](#版本更新与回退)
      - [部分更新资源](#部分更新资源)
      - [编辑资源](#编辑资源)
      - [对资源进行扩缩](#对资源进行扩缩)
      - [删除资源](#删除资源)
      - [与运行中的 Pod 进行交互](#与运行中的-pod-进行交互)
      - [从容器中复制文件和目录](#从容器中复制文件和目录)
      - [与 Deployments 和 Services 进行交互](#与-deployments-和-services-进行交互)
      - [与节点和集群进行交互](#与节点和集群进行交互)
      - [格式化输出](#格式化输出)
      - [端口转发](#端口转发)
      - [调试](#调试)

### 使用 podman-desktop 创建 k8s 集群
1. 安装 podman
2. 安装 podman-desktop
3. 安装 kind
4. 在podman-desktop 安装 kind 扩展
5. Rootful模式重启 podman:
   ```
   podman machine stop
   podman machine set --rootful
   podman machine start
   ```
6. 打开 podman-desktop, settings->resources->kind -> create new

### 使用 kubectl 与集群交互(https://kubernetes.io/zh-cn/docs/reference/kubectl/cheatsheet/)

#### Kubectl 上下文和配置 
```
kubectl config view # 显示合并的 kubeconfig 配置

# 同时使用多个 kubeconfig 文件并查看合并的配置
KUBECONFIG=~/.kube/config:~/.kube/kubconfig2

kubectl config view

# 获取 e2e 用户的密码
kubectl config view -o jsonpath='{.users[?(@.name == "e2e")].user.password}'

kubectl config view -o jsonpath='{.users[].name}'    # 显示第一个用户
kubectl config view -o jsonpath='{.users[*].name}'   # 获取用户列表
kubectl config get-contexts                          # 显示上下文列表
kubectl config current-context                       # 展示当前所处的上下文
kubectl config use-context my-cluster-name           # 设置默认的上下文为 my-cluster-name

kubectl config set-cluster my-cluster-name           # 在 kubeconfig 中设置集群条目

# 在 kubeconfig 中配置代理服务器的 URL，以用于该客户端的请求
kubectl config set-cluster my-cluster-name --proxy-url=my-proxy-url

# 添加新的用户配置到 kubeconf 中，使用 basic auth 进行身份认证
kubectl config set-credentials kubeuser/foo.kubernetes.com --username=kubeuser --password=kubepassword

# 在指定上下文中持久性地保存名字空间，供所有后续 kubectl 命令使用
kubectl config set-context --current --namespace=ggckad-s2

# 使用特定的用户名和名字空间设置上下文
kubectl config set-context gce --user=cluster-admin --namespace=foo \
  && kubectl config use-context gce

kubectl config unset users.foo                       # 删除用户 foo

# 设置或显示 context / namespace 的短别名
# （仅适用于 bash 和 bash 兼容的 shell，在使用 kn 设置命名空间之前要先设置 current-context）
alias kx='f() { [ "$1" ] && kubectl config use-context $1 || kubectl config current-context ; } ; f'
alias kn='f() { [ "$1" ] && kubectl config set-context --current --namespace $1 || kubectl config view --minify | grep namespace | cut -d" " -f6 ; } ; f'
```

#### 创建对象
Kubernetes 配置对象可以用 YAML 或 JSON 定义。可以使用的文件扩展名有 .yaml、.yml 和 .json。
```
kubectl apply -f ./my-manifest.yaml           # 创建资源
kubectl apply -f ./my1.yaml -f ./my2.yaml     # 使用多个文件创建
kubectl apply -f ./dir                        # 基于目录下的所有清单文件创建资源
kubectl apply -f https://git.io/vPieo         # 从 URL 中创建资源
kubectl create deployment nginx --image=nginx # 启动单实例 nginx

# 创建一个打印 “Hello World” 的 Job
kubectl create job hello --image=busybox:1.28 -- echo "Hello World" 

# 创建一个打印 “Hello World” 间隔 1 分钟的 CronJob
kubectl create cronjob hello --image=busybox:1.28   --schedule="*/1 * * * *" -- echo "Hello World"    

kubectl explain pods                          # 获取 Pod 清单的文档说明

# 从标准输入创建多个 YAML 对象
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args:
    - sleep
    - "1000"
EOF

# 创建有多个 key 的 Secret
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo -n "s33msi4" | base64 -w0)
  username: $(echo -n "jane" | base64 -w0)
EOF
```
#### 查看和查找资源
```
# get 命令的基本输出
kubectl get services                          # 列出当前命名空间下的所有 Service
kubectl get pods --all-namespaces             # 列出所有命名空间下的全部的 Pod
kubectl get pods -o wide                      # 列出当前命名空间下的全部 Pod 并显示更详细的信息
kubectl get deployment my-dep                 # 列出某个特定的 Deployment
kubectl get pods                              # 列出当前命名空间下的全部 Pod
kubectl get pod my-pod -o yaml                # 获取一个 Pod 的 YAML

# describe 命令的详细输出
kubectl describe nodes my-node
kubectl describe pods my-pod

# 列出当前名字空间下所有 Service，按名称排序
kubectl get services --sort-by=.metadata.name

# 列出 Pod，按重启次数排序
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 列举所有 PV 持久卷，按容量排序
kubectl get pv --sort-by=.spec.capacity.storage

# 获取包含 app=cassandra 标签的所有 Pod 的 version 标签
kubectl get pods --selector=app=cassandra -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 检索带有 “.” 键值，例如 'ca.crt'
kubectl get configmap myconfig \
  -o jsonpath='{.data.ca\.crt}'

# 检索一个 base64 编码的值，其中的键名应该包含减号而不是下划线
kubectl get secret my-secret --template='{{index .data "key-name-with-dashes"}}'

# 获取所有工作节点（使用选择算符以排除标签名称为 'node-role.kubernetes.io/control-plane' 的结果）
kubectl get node --selector='!node-role.kubernetes.io/control-plane'

# 获取当前命名空间中正在运行的 Pod
kubectl get pods --field-selector=status.phase=Running

# 获取全部节点的 ExternalIP 地址
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个特定 RC 的 Pod 的名称
# 在转换对于 jsonpath 过于复杂的场合，"jq" 命令很有用；可以在 https://jqlang.github.io/jq/ 找到它
sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 显示所有 Pod 的标签（或任何其他支持标签的 Kubernetes 对象）
kubectl get pods --show-labels

# 检查哪些节点处于就绪状态
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"
 
# 使用自定义列检查哪些节点处于就绪状态
kubectl get node -o custom-columns='NODE_NAME:.metadata.name,STATUS:.status.conditions[?(@.type=="Ready")].status'

# 不使用外部工具来输出解码后的 Secret
kubectl get secret my-secret -o go-template='{{range $k,$v := .data}}{{"### "}}{{$k}}{{"\n"}}{{$v|base64decode}}{{"\n\n"}}{{end}}'

# 列出被一个 Pod 使用的全部 Secret
kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq

# 列举所有 Pod 中初始化容器的容器 ID（containerID）
# 可用于在清理已停止的容器时避免删除初始化容器
kubectl get pods --all-namespaces -o jsonpath='{range .items[*].status.initContainerStatuses[*]}{.containerID}{"\n"}{end}' | cut -d/ -f3

# 列出事件（Event），按时间戳排序
kubectl get events --sort-by=.metadata.creationTimestamp

# 列出所有警告事件
kubectl events --types=Warning

# 比较当前的集群状态和假定某清单被应用之后的集群状态
kubectl diff -f ./my-manifest.yaml

# 生成一个句点分隔的树，其中包含为节点返回的所有键
# 在复杂的嵌套JSON结构中定位键时非常有用
kubectl get nodes -o json | jq -c 'paths|join(".")'

# 生成一个句点分隔的树，其中包含为 Pod 等返回的所有键
kubectl get pods -o json | jq -c 'paths|join(".")'

# 假设你的 Pod 有默认的容器和默认的名字空间，并且支持 'env' 命令，可以使用以下脚本为所有 Pod 生成 ENV 变量。
# 该脚本也可用于在所有的 Pod 里运行任何受支持的命令，而不仅仅是 'env'。
for pod in $(kubectl get po --output=jsonpath={.items..metadata.name}); do echo $pod && kubectl exec -it $pod -- env; done

# 获取一个 Deployment 的 status 子资源
kubectl get deployment nginx-deployment --subresource=status
```

#### 版本更新与回退
```
kubectl set image deployment/frontend www=image:v2               # 滚动更新 "frontend" Deployment 的 "www" 容器镜像
kubectl rollout history deployment/frontend                      # 检查 Deployment 的历史记录，包括版本
kubectl rollout undo deployment/frontend                         # 回滚到上次部署版本
kubectl rollout undo deployment/frontend --to-revision=2         # 回滚到特定部署版本
kubectl rollout status -w deployment/frontend                    # 监视 "frontend" Deployment 的滚动升级状态直到完成
kubectl rollout restart deployment/frontend                      # 轮替重启 "frontend" Deployment

cat pod.json | kubectl replace -f -                              # 通过传入到标准输入的 JSON 来替换 Pod

# 强制替换，删除后重建资源。会导致服务不可用。
kubectl replace --force -f ./pod.json

# 为多副本的 nginx 创建服务，使用 80 端口提供服务，连接到容器的 8000 端口
kubectl expose rc nginx --port=80 --target-port=8000

# 将某单容器 Pod 的镜像版本（标签）更新到 v4
kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

kubectl label pods my-pod new-label=awesome                      # 添加标签
kubectl label pods my-pod new-label-                             # 移除标签
kubectl label pods my-pod new-label=new-value --overwrite        # 覆盖现有的值
kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
kubectl annotate pods my-pod icon-                               # 移除注解
kubectl autoscale deployment foo --min=2 --max=10                # 对 "foo" Deployment 自动扩缩容
```

#### 部分更新资源 
```
# 部分更新某节点
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'

# 更新容器的镜像；spec.containers[*].name 是必需的。因为它是一个合并性质的主键。
kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'

# 使用带位置数组的 JSON patch 更新容器的镜像
kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'

# 使用带位置数组的 JSON patch 禁用某 Deployment 的 livenessProbe
kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'

# 在带位置数组中添加元素
kubectl patch sa default --type='json' -p='[{"op": "add", "path": "/secrets/1", "value": {"name": "whatever" } }]'

# 通过修正 scale 子资源来更新 Deployment 的副本数
kubectl patch deployment nginx-deployment --subresource='scale' --type='merge' -p '{"spec":{"replicas":2}}'
```

#### 编辑资源
```
kubectl edit svc/docker-registry                      # 编辑名为 docker-registry 的服务
KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其他编辑器
```

#### 对资源进行扩缩
```
kubectl scale --replicas=3 rs/foo                                 # 将名为 'foo' 的副本集扩缩到 3 副本
kubectl scale --replicas=3 -f foo.yaml                            # 将在 "foo.yaml" 中的特定资源扩缩到 3 个副本
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # 如果名为 mysql 的 Deployment 的副本当前是 2，那么将它扩缩到 3
kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # 扩缩多个副本控制器
```

#### 删除资源 
```
kubectl delete -f ./pod.json                                              # 删除在 pod.json 中指定的类型和名称的 Pod
kubectl delete pod unwanted --now                                         # 删除 Pod 且无宽限期限（无优雅时段）
kubectl delete pod,service baz foo                                        # 删除名称为 "baz" 和 "foo" 的 Pod 和服务
kubectl delete pods,services -l name=myLabel                              # 删除包含 name=myLabel 标签的 Pod 和服务
kubectl -n my-ns delete pod,svc --all                                     # 删除在 my-ns 名字空间中全部的 Pod 和服务
# 删除所有与 pattern1 或 pattern2 awk 模式匹配的 Pod
kubectl get pods  -n mynamespace --no-headers=true | awk '/pattern1|pattern2/{print $1}' | xargs  kubectl delete -n mynamespace pod
```

#### 与运行中的 Pod 进行交互 
```
kubectl logs my-pod                                 # 获取 Pod 日志（标准输出）
kubectl logs -l name=myLabel                        # 获取含 name=myLabel 标签的 Pod 的日志（标准输出）
kubectl logs my-pod --previous                      # 获取上个容器实例的 Pod 日志（标准输出）
kubectl logs my-pod -c my-container                 # 获取 Pod 容器的日志（标准输出, 多容器场景）
kubectl logs -l name=myLabel -c my-container        # 获取含 name=myLabel 标签的 Pod 容器日志（标准输出, 多容器场景）
kubectl logs my-pod -c my-container --previous      # 获取 Pod 中某容器的上个实例的日志（标准输出, 多容器场景）
kubectl logs -f my-pod                              # 流式输出 Pod 的日志（标准输出）
kubectl logs -f my-pod -c my-container              # 流式输出 Pod 容器的日志（标准输出, 多容器场景）
kubectl logs -f -l name=myLabel --all-containers    # 流式输出含 name=myLabel 标签的 Pod 的所有日志（标准输出）
kubectl run -i --tty busybox --image=busybox:1.28 -- sh  # 以交互式 Shell 运行 Pod
kubectl run nginx --image=nginx -n mynamespace      # 在 “mynamespace” 命名空间中运行单个 nginx Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
                                                    # 为运行 nginx Pod 生成规约并将其写入到名为 pod.yaml 的文件

kubectl attach my-pod -i                            # 挂接到一个运行的容器中
kubectl port-forward my-pod 5000:6000               # 在本地计算机上侦听端口 5000 并转发到 my-pod 上的端口 6000
kubectl exec my-pod -- ls /                         # 在已有的 Pod 中运行命令（单容器场景）
kubectl exec --stdin --tty my-pod -- /bin/sh        # 使用交互 shell 访问正在运行的 Pod (一个容器场景)
kubectl exec my-pod -c my-container -- ls /         # 在已有的 Pod 中运行命令（多容器场景）
kubectl top pod POD_NAME --containers               # 显示给定 Pod 和其中容器的监控数据
kubectl top pod POD_NAME --sort-by=cpu              # 显示给定 Pod 的指标并且按照 'cpu' 或者 'memory' 排序
```

#### 从容器中复制文件和目录 
```
kubectl cp /tmp/foo_dir my-pod:/tmp/bar_dir            # 将 /tmp/foo_dir 本地目录复制到远程当前命名空间中 Pod 中的 /tmp/bar_dir
kubectl cp /tmp/foo my-pod:/tmp/bar -c my-container    # 将 /tmp/foo 本地文件复制到远程 Pod 中特定容器的 /tmp/bar 下
kubectl cp /tmp/foo my-namespace/my-pod:/tmp/bar       # 将 /tmp/foo 本地文件复制到远程 “my-namespace” 命名空间内指定 Pod 中的 /tmp/bar
kubectl cp my-namespace/my-pod:/tmp/foo /tmp/bar       # 将 /tmp/foo 从远程 Pod 复制到本地 /tmp/bar
```

#### 与 Deployments 和 Services 进行交互
```
kubectl logs deploy/my-deployment                         # 获取一个 Deployment 的 Pod 的日志（单容器例子）
kubectl logs deploy/my-deployment -c my-container         # 获取一个 Deployment 的 Pod 的日志（多容器例子）

kubectl port-forward svc/my-service 5000                  # 侦听本地端口 5000 并转发到 Service 后端端口 5000
kubectl port-forward svc/my-service 5000:my-service-port  # 侦听本地端口 5000 并转发到 <my-service:my-service-port>

kubectl port-forward deploy/my-deployment 5000:6000       # 侦听本地端口 5000 并转发到 <my-deployment> 创建的 Pod 里的端口 6000
kubectl exec deploy/my-deployment -- ls                   # 在 Deployment 里的第一个 Pod 的第一个容器里运行命令（单容器和多容器例子）
```

#### 与节点和集群进行交互
```
kubectl cordon my-node                                                # 标记 my-node 节点为不可调度
kubectl drain my-node                                                 # 对 my-node 节点进行清空操作，为节点维护做准备
kubectl uncordon my-node                                              # 标记 my-node 节点为可以调度
kubectl top node my-node                                              # 显示给定节点的度量值
kubectl cluster-info                                                  # 显示主控节点和服务的地址
kubectl cluster-info dump                                             # 将当前集群状态转储到标准输出
kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state

# 查看当前节点上存在的现有污点
kubectl get nodes -o='custom-columns=NodeName:.metadata.name,TaintKey:.spec.taints[*].key,TaintValue:.spec.taints[*].value,TaintEffect:.spec.taints[*].effect'

# 如果已存在具有指定键和效果的污点，则替换其值为指定值
kubectl taint nodes foo dedicated=special-user:NoSchedule
```

#### 格式化输出 
输出格式	描述
+ `-o=custom-columns=<spec>`:	使用逗号分隔的自定义列来打印表格
+ `-o=custom-columns-file=<filename>`:	使用 <filename> 文件中的自定义列模板打印表格
+ `-o=go-template=<template>`:	打印在 golang 模板中定义的字段
+ `-o=go-template-file=<filename>`:	打印在 <filename> 文件中由 golang 模板定义的字段
+ `-o=json`:	输出 JSON 格式的 API 对象
+ `-o=jsonpath=<template>`:	打印 jsonpath 表达式中定义的字段
+ `-o=jsonpath-file=<filename>`:	打印在 <filename> 文件中定义的 jsonpath 表达式所指定的字段
+ `-o=name`:	仅打印资源名称而不打印其他内容
+ `-o=wide`:	以纯文本格式输出额外信息，对于 Pod 来说，输出中包含了节点名称
+ `-o=yaml`: 输出 YAML 格式的 API 对象

```
# 集群中运行着的所有镜像
kubectl get pods -A -o=custom-columns='DATA:spec.containers[*].image'

# 列举 default 名字空间中运行的所有镜像，按 Pod 分组
kubectl get pods --namespace default --output=custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"

# 除 "registry.k8s.io/coredns:1.6.2" 之外的所有镜像
kubectl get pods -A -o=custom-columns='DATA:spec.containers[?(@.image!="registry.k8s.io/coredns:1.6.2")].image'

# 输出 metadata 下面的所有字段，无论 Pod 名字为何
kubectl get pods -A -o=custom-columns='DATA:metadata.*'
```

#### 端口转发
转发一个或多个本地端口到某 Pod。

使用资源类型/名称（例如 deployment/mydeployment）来选择 Pod。 如果省略，资源类型默认为 “pod”。

如果有多个 Pod 与条件匹配，将自动选择一个 Pod。 一旦所选 Pod 终止，转发会话也会结束，你需要重新运行命令以恢复转发。

```
kubectl port-forward TYPE/NAME [options] [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N]
```

示例：
```
# 在本地监听端口 5000 和 6000，转发与 Pod 中的端口 5000 和 6000 间的往来数据
kubectl port-forward pod/mypod 5000 6000
  
# 在本地监听端口 5000 和 6000，转发与 Deployment 所选择的 Pod 中端口 5000 和 6000 间往来数据
kubectl port-forward deployment/mydeployment 5000 6000
  
# 在本地监听端口 8443，将数据转发到由 Service 所选择的 Pod 中名为 "https" 的服务端口的 targetPort
kubectl port-forward service/myservice 8443:https
  
# 在本地监听端口 8888，将数据转发到 Pod 中的端口 5000
kubectl port-forward pod/mypod 8888:5000
  
# 在所有地址上监听端口 8888，将数据转发到 Pod 中的端口 5000
kubectl port-forward --address 0.0.0.0 pod/mypod 8888:5000
  
# 在 localhost 和选定的 IP 上监听端口 8888，将数据转发到 Pod 中的端口 5000
kubectl port-forward --address localhost,10.19.21.23 pod/mypod 8888:5000
  
# 在本地监听随机端口，将数据转发到 Pod 中的端口 5000
kubectl port-forward pod/mypod :5000
```

#### 调试
```
kubectl proxy & // 开启代理转发后可以使用 curl 直接访问 apiserver 的接口
curl http://127.0.0.1:8001/apis/samplecontroller.k8s.io/v1alpha1/namespaces/default/foos?watch=true
```