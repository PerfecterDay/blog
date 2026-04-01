# K8S 更新策略
{docsify-updated}

> https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/update/update-intro/

Kubernetes 的 Deployment 控制器内置了强大的滚动更新机制（Rolling Update），用于平滑地将旧版本 Pod 替换为新版本，同时保持服务可用性。  
在 Kubernetes 中，更新是具有版本控制的，任何 Deployment 更新都可以恢复到以前的（稳定）版本。

Deployment 支持两种主要更新策略：
+ `RollingUpdate` （默认）: 滚动更新，逐步替换旧 Pod，保持服务不中断
+ `Recreate` : 删除所有旧 Pod 后再启动新 Pod（会短暂中断服务）

## RollingUpdate
滚动更新通过增量式更新 Pod 实例并替换为新的实例，允许在 Deployment 更新过程中实现零停机。如果 Deployment 的访问是公开的，Service 在更新期间仅将流量负载均衡到可用的 Pod。

滚动更新一次仅更新一批Pod(数量取决于yaml中的配置)，当更新的Pod就绪后再更新另一批，直到全部更新完成为止；该策略实现了不间断服务的目标，但是在更新过程中，不同客户端得到的响应内容可能会来自不同版本的应用。会出现新老版本共存状态。

```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```
1. `maxSurge` : 可超出期望副本数的最大 Pod 数，控制新增 Pod 数量。可以是绝对值或百分比，如 1 或 25%
2. `maxUnavailable` : 更新过程中允许不可用的 Pod 数量。控制旧 Pod 最多同时 down 几个。也可为百分比


以副本数 3、maxSurge=1、maxUnavailable=1 为例：
```
初始状态：3 个旧 Pod 运行中
➜ 新版本镜像发布

Step 1:
+ 创建新 Pod（1个） → 4 Pod 总数
- 停止 1 个旧 Pod       → 保持 maxUnavailable=1

Step 2:
+ 再创建一个新 Pod
- 再停止一个旧 Pod

Step 3:
+ 再创建一个新 Pod（第 3 个新）
- 停止最后一个旧 Pod

最终完成替换
```

## k8s 执行更新的操作步骤
1. `kubectl set image deployment/myapp myapp=myapp:v2`: 更新 deployment 的镜像版本
2. `kubectl rollout restart deployment myapp` : 更新
3. `kubectl rollout status deployment myapp` : 监控更新状态
4. `kubectl rollout history deployment myapp` : 查看历史版本
5. `kubectl rollout undo deployment myapp` : 回退到上一版本
6. `kubectl rollout undo deployment myapp --to-revision=3` : 回滚到指定历史版本