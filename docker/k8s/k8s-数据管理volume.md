# k8s 数据持久化管理-volume
{docsify-updated}

容器和Pod是短暂的。其含义是它们的生命周期可能很短，会被频繁地销毁和创建。容器销毁时，保存在容器内部文件系统中的数据都会被清除。为了持久化保存容器的数据，可以使用Kubernetes Volume。

Volume的生命周期独立于容器，Pod中的容器可能被销毁和重建，但Volume会被保留。**本质上，Kubernetes Volume是一个目录，这一点与Docker Volume类似**。当Volume被mount到Pod，Pod中的所有容器都可以访问这个Volume。Kubernetes Volume也支持多种backend类型,emptyDir,hostPath.GCE Persistent Disk,AWS Elastic Block Store,NFS、Ceph等。

Volume提供了对各种backend的抽象，容器在使用Volume读写数据的时候不需要关心数据到底是存放在本地节点的文件系统中还是云硬盘上。**对它来说，所有类型的Volume都只是一个目录**。

## emptyDir
emptyDir 是最基础的Volume 类型。正如其名字所示， **一个emptyDir Volume 是 Host 上的一个空目录。**  
emptyDir Volume 对于容器来说是持久的，对于Pod则不是。当Pod 从节点删除时， Volume 的内容也会被删除。但如果只是容器被销毁而 Pod 还在，则 Volume 不受影响。   
也就是说: emptyDir Volume 的生命周期与Pod一致。

```
apiVersion: v1 
kind:Pod 
metadata:
  name:producer-consumer 
spec:
  containers:
  - image: busybox
    name: producer 
    volumeMounts:
    - mountPath: /producer_dir 
      name:shared-volume
    args:
    - /bin/sh
    - -c
    - echo "hello world" > /producer_dir/hello; sleep 30000

  - image: busybox
    name: consumer 
    volumeMounts:
    - mountPath: /consumer_dir 
      name:shared-volume
    args:
    - /bin/sh
    - -c
    - cat /consumer_dir/hello; sleep 30000
  
  volumes:
  - name: shared-volume
    emptyDir: {}
```

Pod 中的所有容器都可以共享 Volume，它们可以指定各自的 mount 路径。因为emptyDir 是 Docker Host 文件系统里的目录，其效果相当于执行了`docker run -v /producer_dir` 和`docker run -v /consumer _dir`。通过dockerinspect 查看容器的详细配置信息， 我们发现两个容器都 mount 了同一个目录:
```
"Mounts": [
{
    "Type": "bind",
    "Source": "/var/lib/kubelet/pods/4021d63e-8a21-4cd1-99e9-76f273d3736b/volumes/kubernetes.io~empty-dir/shared-volume",
    "Destination": "/consumer_dir",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
}

"Mounts": [
{
    "Type": "bind",
    "Source": "/var/lib/kubelet/pods/4021d63e-8a21-4cd1-99e9-76f273d3736b/volumes/kubernetes.io~empty-dir/shared-volume",
    "Destination": "/producer_dir",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
}
```
这里的 `/var/lib/kubelet/pods/4021d63e-8a21-4cd1-99e9-76f273d3736b/volumes/kubernetes.io~empty-dir/shared-volume` 就是真实主机上的路径。

emptyDir 是Host 上创建的临时目录，其优点是能够方便地为Pod 中的容器提供共享存储，不需要额外的配置。它不具备持久性，如果Pod 不存在了，emptyDir 也就没有了。根据这个特性，emptyDir 特别适合 Pod 中的容器需要临时共享存储空间的场景，比如前面的生产者、消费者用例。

## hostPath
hostPath Volume 的作用是**将 Docker Host 文件系统中己经存在的目录 mount 给 Pod 的容器**。大部分应用都不会使用hostPath Volume，因为这实际上增加了Pod 与Host 主机节点的耦合， 限制了Pod 的使用。不过那些需要访问 Kubernetes 或 Docker 内部数据 (配置文件和二进制库)的应用则需要使用 hostPath。

如果Pod被销毁了，hostPath 对应的目录还是会被保留，从这一点来看，hostPath 的持久性比 emptyDir 强。不过一旦Host崩溃，hostPath 也就无法访问了。

## 外部Storage Provider
如果Kubernetes 部署在诸如AWS、GCE、Azure 等公有云上，可以直接使用云硬盘作 为Volume。  
相对 于emptyDir 和 hostPath，这些 Volume 类型的最大特点就是不依赖 Kubernetes。 Volume 的底层基础设施由独立的存储系统管理，与Kubernetes 集群是分离的。数据被持久 化后，即使整个Kubernetes 崩溃也不会受损。

## PersistentVolume 和 PersistentVolumeClaim
Volume 提供 了非常好的数据持久化方案，不过在可管理性上还有不足。 拿前面的AWSEBS例子来说，要使用Volume，Pod 必须事先知道如下信息: 
1. 当前Volume 来自AWS EBS。
2. EBS Volume 已经提前创建，并且知道确切的volume-id。

Pod通常是由应用的开发人员维护，而Volume则通常是由存储系统的管理员维护。开发人员要获得上面的信息，要么询问管理员，要么自己就是管理员。这样就带来一个管理上的问题:应用开发人员和系统管理员的职责耦合在一起了。如果系统规模较小或者对于开发环境，这样的情况还可以接受，当集群规模变大，特别是对于生产环境，考虑到效率和安全性，这就成了必须要解决的问题。

Kubernetes 给出的解决方案是 `PersistentVolume` 和 `PersistentVolumeClaim` 。  
`PersistentVolume` (PV) 是外部存储系统中的一块存储空间，由管理员创建和维护。与 Volume 一样，PV 具有持久性，生命周期独立于Pod。 持久卷是集群资源，就像节点也是集群资源一样。

`PersistentVolumeClaim` (PVC)是对 PV 的申请 (Claim)，表达的是用户对存储的请求。PVC 通常由普通用户创建和维护。需要为Pod 分配存储资源时，用户可以创建一个PVC，指明存储资源的容量大小和访问模式 (比如只读)等信息，Kubernetes 会查找并提供满足条件的 PV。 

有了 `PersistentVolumeClaim` ，用户只需要告诉Kuberetes 需要什么样的存储资源，而不必关心真正的空间从哪里分配、如何访问等底层细节信息。这些 Storage Provider 的底层信息 交给管理员来处理，只有管理员才应该关心创建 PersistentVolume 的细节信息。

Kubernetes 支持多种类型的 PersistentVolume，比如 AWS EBS、Ceph、NFS 等。

### PV
PV 卷的制备有两种方式：静态制备或动态制备。

1. 静态制备  
集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息， 并且对集群用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）。

2. 动态制备  
如果管理员所创建的所有静态 PV 卷都无法与用户的 `PersistentVolumeClaim` 匹配， 集群可以尝试为该 PVC 申领动态制备一个存储卷。 这一制备操作是基于 `StorageClass` 来实现的：PVC 申领必须请求某个存储类， 同时集群管理员必须已经创建并配置了该类，这样动态制备卷的动作才会发生。 如果 PVC 申领指定存储类为 ""，则相当于为自身禁止使用动态制备的卷。

为了基于存储类完成动态的存储制备，集群管理员需要在 API 服务器上启用 `DefaultStorageClass` 准入控制器。 举例而言，可以通过保证 `DefaultStorageClass` 出现在 API 服务器组件的 `--enable-admission-plugins` 标志值中实现这点；该标志的值可以是逗号分隔的有序列表。

3. 绑定  
用户创建一个带有特定存储容量和特定访问模式需求的 `PersistentVolumeClaim` 对象； 在动态制备场景下，这个 PVC 对象可能已经创建完毕。 控制平面中的控制回路监测新的 PVC 对象，寻找与之匹配的 PV 卷（如果可能的话）， 并将二者绑定到一起。 如果为了新的 PVC 申领动态制备了 PV 卷，则控制回路总是将该 PV 卷绑定到这一 PVC 申领。 否则，用户总是能够获得他们所请求的资源，只是所获得的 PV 卷可能会超出所请求的配置。 一旦绑定关系建立，则 `PersistentVolumeClaim` 绑定就是排他性的， 无论该 PVC 申领是如何与 PV 卷建立的绑定关系。 PVC 申领与 PV 卷之间的绑定是一种一对一的映射，实现上使用 ClaimRef 来记述 PV 卷与 PVC 申领间的双向绑定关系。

如果找不到匹配的 PV 卷，PVC 申领会无限期地处于未绑定状态。 当与之匹配的 PV 卷可用时，PVC 申领会被绑定。 例如，即使某集群上制备了很多 50 Gi 大小的 PV 卷，也无法与请求 100 Gi 大小的存储的 PVC 匹配。当新的 100 Gi PV 卷被加入到集群时， 该 PVC 才有可能被绑定。

### PV实例
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv1
spec:
  capacity:
    storage: 1Gi
    accessModes: 
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle 
  storageClassName: nfs
  nfs:
    path: /nfsdata/pv1
    server: 192.168.56.10
```
+ `capacity` 指定PV 的容量为 1GB。
+ `accesModes` 指定访问模式为 `ReadWriteOnce` ，支持的访问模式有3种:
  + ReadWriteOnce 表示PV能以read-write 模式mount 到单个节点，
  + ReadOnlyMany 表示PV能以read-only 模式 mount 到多个节点，
  + ReadWriteMany 表示PV 能以 read -write 模式 mount 到多个节点。 
+ `persistentVolumeReclaimPolicy` 指定当PV的回收策略为Recycle，支持的策略有3种:
  + `Retain`表示需要管理员手工回收:
  + `Recycle` 表示清除PV中的数据，效果相当于执行 `rm -rf /thevolume/*`
  + `Delete` 表示删除 StorageProvider 上的对应存储资源，例如AWS EBS、GCE PD. Azure Disk, OpenStack Cinder Volume 等
+ `storageClassName` 指定PV的class 为nfs。相当于次PV设置了一个分类，PVC可 以指定class 申清相座class 的PV。
+ `path`指定PV在NFS服务器上对应的目录。

### pvc

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-consul-consul-consul-server-0
  namespace: consul
  resourceVersion: '208713526'
  uid: 9582b0e7-b5a2-4a74-a479-dd3f62eb37cf
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nas
  volumeMode: Filesystem
  volumeName: consul-server02
status:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  phase: Bound
```

然后需要在pod中使用的时候直接 mount 到相应路径即可：

```yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: consul
    statefulset.kubernetes.io/pod-name: consul-consul-server-0
    name: consul-consul-server-0
  namespace: consul
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - podAffinityTerm:
            labelSelector:
              matchLabels:
                app: consul
                component: server
                release: consul
            topologyKey: topology.kubernetes.io/zone
          weight: 100
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: consul
              component: server
              release: consul
          topologyKey: kubernetes.io/hostname
  containers:
    - command:
        - /bin/sh
      image: 'demo-registry-registry-vpc.cn-hongkong.cr.aliyuncs.com/sit/consul:1.15.1'
      imagePullPolicy: IfNotPresent
      name: consul
      ports:
        - containerPort: 8500
          name: http
          protocol: TCP
      volumeMounts:
        - mountPath: /consul/data
          name: data-consul
        - mountPath: /consul/config
          name: config
        - mountPath: /consul/extra-config
          name: extra-config
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-m9rc7
          readOnly: true
  volumes:
    - name: data-consul
      persistentVolumeClaim:
        claimName: data-consul-consul-consul-server-0
    - configMap:
        defaultMode: 420
        name: consul-consul-server-config
      name: config
    - emptyDir: {}
      name: extra-config
    - name: kube-api-access-m9rc7
      projected:
        defaultMode: 420
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
              path: token
          - configMap:
              items:
                - key: ca.crt
                  path: ca.crt
              name: kube-root-ca.crt
          - downwardAPI:
              items:
                - fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
                  path: namespace
status:
  conditions:
    - lastTransitionTime: '2024-11-16T13:25:36Z'
      status: 'True'
      type: Initialized
    - lastTransitionTime: '2024-11-16T13:25:49Z'
      status: 'True'
      type: Ready
    - lastTransitionTime: '2024-11-16T13:25:49Z'
      status: 'True'
      type: ContainersReady
    - lastTransitionTime: '2024-11-16T13:25:36Z'
      status: 'True'
      type: PodScheduled
  containerStatuses:
    - containerID: >-
        containerd://104337b269264cf28a9501a0499295dee585b21ec11e72a25a93de91b6daa953
      image: 'demo-registry-registry-vpc.cn-hongkong.cr.aliyuncs.com/sit/consul:1.15.1'
      imageID: >-
        demo-registry-registry-vpc.cn-hongkong.cr.aliyuncs.com/sit/consul@sha256:f7d7f31289176687415ec29f0c663b39fb7c30c950852040fd7d589d5f82da88
      lastState: {}
      name: consul
      ready: true
      restartCount: 0
      started: true
      state:
        running:
          startedAt: '2024-11-16T13:25:37Z'
  hostIP: 10.4.153.119
  phase: Running
  podIP: 10.4.153.15
  podIPs:
    - ip: 10.4.153.15
  qosClass: Guaranteed
  startTime: '2024-11-16T13:25:36Z'
```