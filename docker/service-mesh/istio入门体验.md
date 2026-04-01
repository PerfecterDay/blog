#Istio
{docsify-updated}

> https://istio.io/latest/zh/docs/setup/getting-started/

- [安装 istio](#%E5%AE%89%E8%A3%85%20istio)
- [windows 部署 bookinfo](#windows%20%E9%83%A8%E7%BD%B2%20bookinfo)
- [初体验流量管理](#%E5%88%9D%E4%BD%93%E9%AA%8C%E6%B5%81%E9%87%8F%E7%AE%A1%E7%90%86)
	- [路由到版本1](#%E8%B7%AF%E7%94%B1%E5%88%B0%E7%89%88%E6%9C%AC1)
	- [基于用户身份的路由](#%E5%9F%BA%E4%BA%8E%E7%94%A8%E6%88%B7%E8%BA%AB%E4%BB%BD%E7%9A%84%E8%B7%AF%E7%94%B1)
### 安装 istio
1. 安装 istioctl 工具：
	```
	brew install istioctl --mac
	scoop install istioctl --windows
	```
2. 在k8s集群中安装 istio
	```
	istioctl install --set profile=demo -y
	kubectl label namespace default istio-injection=enabled //给命名空间添加标签，指示 Istio 在部署应用的时候，自动注入 Envoy 边车代理
	```

### windows 部署 bookinfo
1. clone github 代码：`git clone git@github.com:istio/istio.git`
2. `kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml` --部署 bookinfo 
3. `kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | findstr -o "<title>.*</title>"` --验证
4. 创建 Istio 入站网关（Ingress Gateway），它会在网格边缘把一个路径映射到路由: `kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml`
5. 将电脑的 8080 端口映射到 istio-ingressgateway 服务的 80 端口：`kubectl port-forward svc/istio-ingressgateway 8080:80 -n istio-system`
6. 访问： http://localhost:8080/productpage
7. 安装扩展：`kubectl apply -f samples/addons`
8. 安装 kiali: `kubectl rollout status deployment/kiali -n istio-system`
9. 访问kaili： `istioctl dashboard kiali`

### 初体验流量管理

Istio [Bookinfo](https://istio.io/latest/zh/docs/examples/bookinfo/) 示例包含四个独立的微服务， 每个微服务都有多个版本。其中一个微服务 `reviews` 的三个不同版本已经部署并同时运行。 为了说明这导致的问题，在浏览器中访问 Bookinfo 应用程序的 `/productpage` 并刷新几次。您会注意到，有时书评的输出包含星级评分，有时则不包含。这是因为没有明确的默认服务版本可路由， Istio 将以循环方式将请求路由到所有可用版本。

此任务的最初目标是应用将所有流量路由到微服务的 `v1` （版本 1）的规则。稍后，您将应用规则根据 HTTP 请求 header 的值路由流量。

#### UI界面修改路由比例
<center><img src="pics/kali-1.jpg" width="60%"></center>

#### 路由到版本1
1. 在可以使用 Istio 控制 Bookinfo 版本路由之前，需要定义可用的版本：
	```
	kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
	```
2.  `kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`

现在无论刷新多少次，页面的评论部分都不会显示评级星标。这是因为您将 Istio 配置为将评论服务的所有流量路由到版本 `reviews:v1`，而此版本的服务不访问星级评分服务。

#### 基于用户身份的路由
更改路由配置，以便将来自特定用户的所有流量路由到特定服务版本。在这种情况下， 来自名为 Jason 的用户的所有流量将被路由到服务 `reviews:v2`。

Istio 对用户身份没有任何特殊的内置机制。事实上，`productpage` 服务在所有到 `reviews` 服务的 HTTP 请求中都增加了一个自定义的 `end-user` 请求头，从而达到了本例子的效果。

`kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml`

在 Bookinfo 应用程序的 `/productpage` 上，以用户 `jason` 身份登录。
    
    刷新浏览器。您看到了什么？星级评分显示在每个评论旁边。
    
以其他用户身份登录（选择您想要的任何名称）。
    
    刷新浏览器。现在星星消失了。这是因为除了 Jason 之外，所有用户的流量都被路由到 `reviews:v1`。