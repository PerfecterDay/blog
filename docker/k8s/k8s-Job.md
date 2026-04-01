# k8s-Job和定时任务 CronJob
{docsify-updated}

- [k8s-Job和定时任务 CronJob](#k8s-job和定时任务-cronjob)
	- [CronJob](#cronjob)

容器按照持续运行的时间可分为两类：服务类容器和工作类容器。

服务类容器通常持续提供服务，需要一直运行，比如 http server，daemon 等。工作类容器则是一次性任务，比如批处理程序，完成后容器就退出。

Kubernetes 的 Deployment、ReplicaSet 和 DaemonSet 都用于管理服务类容器；对于工作类容器，我们用 Job。


当第一个 Pod 启动时，容器失败退出，根据 restartPolicy: Never，此失败容器不会被重启，但 Job DESIRED 的 Pod 是 1，目前 SUCCESSFUL 为 0，不满足，所以 Job controller 会启动新的 Pod，直到 SUCCESSFUL 为 1。对于我们这个例子，SUCCESSFUL 永远也到不了 1，所以 Job controller 会一直创建新的 Pod。为了终止这个行为，只能删除 Job。

如果将 restartPolicy 设置为 OnFailure 会怎么样
这里只有一个 Pod，不过 RESTARTS 为 3，而且不断增加，说明 OnFailure 生效，容器失败后会自动重启。

有时，我们希望能同时运行多个 Pod，提高 Job 的执行效率。这个可以通过 parallelism 设置。

我们还可以通过 completions 设置 Job 成功完成 Pod 的总数：
```
parallelism: 2
completions: 6
```
上面配置的含义是：每次运行两个 Pod，直到总共有 6 个 Pod 成功完成


### CronJob
Linux 中有 cron 程序定时执行任务，Kubernetes 的 CronJob 提供了类似的功能，可以定时执行 Job。将 kind 设置为 CronJob。
