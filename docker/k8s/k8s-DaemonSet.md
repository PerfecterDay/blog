# k8s DaemonSet
{docsify-updated}

- [k8s DaemonSet](#k8s-daemonset)
	- [kube-proxy](#kube-proxy)

Deployment 部署的副本 Pod 会分布在各个 Node 上，每个 Node 都可能运行好几个副本。DaemonSet 的不同之处在于：**每个 Node 上最多只能运行一个副本**。

DaemonSet 的典型应用场景有：
+ 在集群的每个节点上运行存储 Daemon，比如 glusterd 或 ceph。
+ 在每个节点上运行日志收集 Daemon，比如 flunentd 或 logstash。
+ 在每个节点上运行监控 Daemon，比如 Prometheus Node Exporter 或 collectd。

其实 Kubernetes 自己就在用 DaemonSet 运行系统组件。执行如下命令：
`kubectl get daemonset --namespace=kube-system`
可以看到 DaemonSet kube-flannel-ds 和 kube-proxy 分别负责在每个节点上运行 flannel 和 kube-proxy 组件。 

1. DaemonSet 配置文件的语法和结构与 Deployment 几乎完全一样，只是将 kind 设为 DaemonSet。
2. hostNetwork 指定 Pod 直接使用的是 Node 的网络，相当于 `docker run --network=host`。考虑到 flannel 需要为集群提供网络连接，这个要求是合理的。
3. containers 定义了运行 flannel 服务的两个容器。


### kube-proxy
`kubectl edit daemonset kube-proxy --namespace=kube-system`

```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2023-12-23T06:48:34Z"
  generation: 1
  labels:
    k8s-app: kube-proxy
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "432"
  uid: 01cc3598-29a4-4443-9def-2d5f0c531e6d
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-proxy
    spec:
      containers:
      - command:
        - /usr/local/bin/kube-proxy
        - --config=/var/lib/kube-proxy/config.conf
        - --hostname-override=$(NODE_NAME)
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: registry.k8s.io/kube-proxy:v1.27.3
        imagePullPolicy: IfNotPresent
        name: kube-proxy
        resources: {}
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/kube-proxy
          name: kube-proxy
        - mountPath: /run/xtables.lock
          name: xtables-lock
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-node-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: kube-proxy
      serviceAccountName: kube-proxy
      terminationGracePeriodSeconds: 30
      tolerations:
      - operator: Exists
      volumes:
      - configMap:
          defaultMode: 420
          name: kube-proxy
        name: kube-proxy
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock
      - hostPath:
          path: /lib/modules
          type: ""
        name: lib-modules
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
status:
  currentNumberScheduled: 1
  desiredNumberScheduled: 1
  numberAvailable: 1
  numberMisscheduled: 0
  numberReady: 1
  observedGeneration: 1
  updatedNumberScheduled: 1
```