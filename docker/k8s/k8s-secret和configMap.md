# k8s-secret和configMap
{docsify-updated}

## Secret
应用启动过程中可能需要一些敏感信息，比如访问数据库的用户名、密码或者密钥。将这 些信息直接保存在容器镜像中显然不妥，Kubernetes 提供的解决方案是Secret。
Secret 会以密文的方式存储数据，避免了直接在配置文件中保存敏感信息。POD 服务有两种方式访问这些数据：
+ Secret 会以 Volume 的形式被 mount 到Pod，会以文件的形式存在于挂载的容器的挂载路径中，容器应用可以以**访问文件**的方式使用 Secret 中的敏感数据;
+ 容器也可以以**环境变量**的方式使用这些数据。

Secret 可通过命令行或 YAML 创建。

#### 创建 secret
1. `--from-literal`
   ```
    kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=123456
   ```
2. `--from-file`
   ```
    echo -n admin > ./username
    echo -n 123456 > ./password
    kubectl create secret generic mysecret --from-file=./username --from-file=./password
   ```
3. `-from-env-file`
   ```
    cat << EOF > env.txt
    username=admin
    password=123456
    EOF
    kubectl create secret generic mysecret --from-env-file=env.txt
   ```
4. YAML配置文件
   ```
   apiVersion: v1 
   kind: Secret 
   metadata:
     name: mysecret 
     data:
       username: YWRtaW4= 
       password: MTIzNDU2
   ```

### 查看 secret
1. `kubectl get secret <secret_name>`
2. `kubectl describe secret <secret_name>` ： 查看指定 secret 包含的 key
3. `kubectl edit secret <secret_name>` : 查看指定 secret 的 value


### pod 中使用 secret
Pod 可以通过Volume 或者环境变量的方式使用Secret。

#### Volume 方式
```
apiVersion: v1 
kind: Pod
metadata:
  name: mypod 
spec:
  containers:
  - name: mypod 
    image: busybox
    args: 
      - /bin/sh
      - -С
      - sleep 10; touch /tmp/healthy; sleep 300000
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes: 
  - name: foo
    secret: 
      secretName: mysecret 
      items:
      - key: username
        path: my-group/my-username
      - key: password
        path: my-group/my-password
```
可以自定义存放数据的文件名，如上所示，这时数据将分别存放在 `/etc/foo/my-group/my-username` 和 `/etc/foo/my-group/my-password`中。
Kubernetes 默认会在指定的路径 `/etc/foo` 下为每条敏感数据创建一个文件，文件名默认就是数据条目的 Key， Value 则以明文存放在文件中。

**以 Volume 方式使用的 Secret 支持动态更新: Secret 更新后，容器中的数据也会更新。**

#### 环境变量方式
通过 Volume 使用 Secret，容器必须从文件读取数据，稍显麻烦，Kubernetes 还支持通过环境变量使用Secret
```
apiVersion: v1 
kind: Pod
metadata:
  name: mypod 
spec:
  containers:
  - name: mypod 
    image: busybox
    args: 
      - /bin/sh
      - -С
      - sleep 10; touch /tmp/healthy; sleep 3000
    env:
      - name: SECRET_USERNAME
        valueFrom: 
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom: 
          secretKeyRef:
            name: mysecret
            key: password
```
`SECRET_USERNAME`/`SECRET_PASSWORD` 可以作为环境变量名被应用使用。

需要注意的是，**环境变量读取Secret 很方便，但无法支撑Secret 动态更新。**

## ConfigMap
Secret 可以为Pod 提供密码、Token、私钥等敏感数据;对于一些非敏感数据，比如应 用的配置信息，则可以用 ConfigMap。与 Secret 类似，POD 也可以通过 volume 挂载或者环境变量两种方式使用 ConfigMap 。

```
apiVersion: v1 
kind: ConfigMap 
metadata:
    name: myConfigMap
    data:
    username: YWRtaW4= 
    password: MTIzNDU2
```


volume方式：
```
apiVersion: v1 
kind: Pod
metadata:
  name: mypod 
spec:
  containers:
  - name: mypod 
    image: busybox
    args: 
      - /bin/sh
      - -С
      - sleep 10; touch /tmp/healthy; sleep 300000
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes: 
  - name: foo
    configMap: 
      name: myConfigMap 
      items:
      - key: username
        path: my-group/my-username
      - key: password
        path: my-group/my-password
```

env方式：
```
apiVersion: v1 
kind: Pod
metadata:
  name: mypod 
spec:
  containers:
  - name: mypod 
    image: busybox
    args: 
      - /bin/sh
      - -С
      - sleep 10; touch /tmp/healthy; sleep 3000
    env:
      - name: CONFIG_USERNAME
        valueFrom: 
          configMapKeyRef:
            name: myConfigMap
            key: username
      - name: CONFIG_PASSWORD
        configMapKeyRef: 
          secretKeyRef:
            name: myConfigMap
            key: password
```

### 阿里云 ConfigMap 实战
1. 新建 configMap(nts-config)，名称(key) 为配置文件名: nts-sit.xml，后续deployment 挂载的时候，会以key 的名字在挂载路径下生成配置文件
<center><img src="pics/ali-configmap-1.png" width="40%"></center>

2. 部署deployment 的时候，挂载名字为 nts-config 的 configmap 到指定路径
<center><img src="pics/ali-configmap-2.png" width="40%"></center>

3. 修改代码加载配置文件的路径到挂载的路径
<center><img src="pics/ali-configmap-3.png" width="40%"></center>

可以看到部署的容器内部，会在挂载路径下生成 configMap 中的文件，文件名是配置时key 的名字，内容就是配置的内容：
<center><img src="pics/ali-configmap-4.png" width="40%"></center>

configMap 的配置在 K8S 中更新后会实时反映在挂载的 deployment 中，也就是说相应 pod 中挂载的文件会实时更新反映 configMap 的更新，而不需要重新部署 deployment。