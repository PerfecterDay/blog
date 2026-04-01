# Ingress 和 gateway api
{docsify-updated}

- [Ingress 和 gateway api](#ingress-和-gateway-api)
  - [Ingress 和 Service 的区别](#ingress-和-service-的区别)
  - [gateway api](#gateway-api)
    - [与 Ingress的区别](#与-ingress的区别)


Ingress 公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

## Ingress 和 Service 的区别
由于您只有一个提供 http 服务的服务，因此您目前使用 LoadBalancer 服务类型的解决方案效果很好。试想一下，你有多个基于 http 的服务，你想让它们在不同的路由上对外提供服务。您必须为每个服务创建一个 LoadBalancer 服务，默认情况下，您将为每个服务获得不同的 IP 地址。相反，您可以使用 Ingress，它位于这些服务之前并负责路由选择。
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

这样的话，对外部调用方来说，只要调用暴露的这个ingress端点即可。由 ingress 去做路由分发到不通服务。

<center><img src="/pics/ingressFanOut.svg" width="60%"></center>

## gateway api
Gateway API 是 Ingress API 的后续版本。不过，它不包括 Ingress 类型。

### 与 Ingress的区别
> https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/

