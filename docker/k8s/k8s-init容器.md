# Init 容器
{docsify-updated}

每个 Pod 中可以包含多个容器， 应用运行在这些容器里面，同时 Pod 也可以**有一个或多个先于应用容器启动的 Init 容器**。

Init 容器与普通的容器非常像，除了如下两点：
+ 它们总是运行到完成。
+ 每个都必须在下一个启动之前成功完成。

如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止。 然而，如果 Pod 对应的 restartPolicy 值为 "Never"，并且 Pod 的 Init 容器失败， 则 Kubernetes 会将整个 Pod 状态设置为失败。

## 与普通容器的不同之处
Init 容器支持应用容器的全部字段和特性，包括资源限制、 数据卷和安全设置。 然而，Init 容器对资源请求和限制的处理稍有不同， 在下面容器内的资源共享节有说明。

常规的 Init 容器（即不包括边车容器）**不支持** `lifecycle、livenessProbe、readinessProbe` 或 `startupProbe` 字段。Init 容器必须在 Pod 准备就绪之前完成运行；而边车容器在 Pod 的生命周期内继续运行， 它支持一些探针。

如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。 每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时， Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

## 与边车容器的不同之处
+ Init 容器在主应用容器启动之前运行并完成其任务。 与边车容器不同， Init 容器不会持续与主容器一起运行。
+ Init 容器按顺序完成运行，等到所有 Init 容器成功完成之后，主容器才会启动。
+ Init 容器不支持 lifecycle、livenessProbe、readinessProbe 或 startupProbe， 而边车容器支持所有这些探针以控制其生命周期。
+ Init 容器与主应用容器共享资源（CPU、内存、网络），但不直接与主应用容器进行交互。 不过这些容器可以使用共享卷进行数据交换。

