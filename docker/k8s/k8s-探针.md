# k8s 探针
{docsify-updated}

> https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

## 存活探针(liveness probe)-探活失败重启POD
存活探针决定何时重启容器。 例如，当应用在运行但死锁变得无响应时，存活探针可以捕获这类死锁。如果一个容器的存活探针失败多次，kubelet 将**重启该容器**。注意，是重启容器，不是删除重建，K8s 阿里云上看，pod name id 不会变，只是重启次数增加了。  

kubelet 会向容器内运行的服务（比如服务在监听 8080 端口）发送一个 `HTTP GET` 请求来执行探测。 如果服务器返回成功代码，则 kubelet 认为容器是健康存活的。 如果处理程序返回失败代码，则 kubelet 会杀死这个容器并将其重启。

**服务器返回大于或等于 200 并且小于 400 的任何代码都标示成功，其它返回代码都标示失败。**

**存活探针不会等待就绪探针成功**。 如果你想在执行存活探针前等待，你可以定义 `initialDelaySeconds` ，或者使用启动探针。

### GRPC 存活探针
```
apiVersion: v1
kind: Pod
metadata:
  name: etcd-with-grpc
spec:
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.1-0
    command: [ "/usr/local/bin/etcd", "--data-dir",  "/var/lib/etcd", "--listen-client-urls", "http://0.0.0.0:2379", "--advertise-client-urls", "http://127.0.0.1:2379", "--log-level", "debug"]
    ports:
    - containerPort: 2379
    livenessProbe:
      grpc:
        port: 2379
      initialDelaySeconds: 10
```


## 就绪探针(readiness probe)-就绪探测失败就禁止流量转发
**就绪探针决定容器何时准备好接受流量**。 这种探针在等待应用执行耗时的初始任务时非常有用； 例如：建立网络连接、加载文件和预热缓存。

在容器的生命周期后期， 就绪探针也很有用，例如，从临时故障或过载中恢复时。

如果就绪探针返回的状态为失败，**Kubernetes 会将该 Pod 从所有对应服务的端点中移除**。

就绪探针在容器的整个生命期内**持续运行**。

就绪探针的配置和存活探针的配置相似。 唯一区别就是要使用 `readinessProbe` 字段，而不是 `livenessProbe` 字段。

## 启动探针(startup probe)-启动成功之前不进行就绪探测和探活检测
启动探针检查容器内的应用是否已启动。 **启动探针可以用于对慢启动容器进行存活性检测，避免它们在启动运行之前就被 kubelet 杀掉。如果配置了这类探针，它会禁用存活检测和就绪检测，直到启动探针成功为止。**

这类探针**仅在启动时执行**，不像存活探针和就绪探针那样周期性地运行。

## 综合示例
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    args:
    ports:
    - name: liveness-port
      containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: liveness-port
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      failureThreshold: 1
      initialDelaySeconds: 3
      periodSeconds: 3
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /healthz
        port: liveness-port
      failureThreshold: 30
      periodSeconds: 10
```