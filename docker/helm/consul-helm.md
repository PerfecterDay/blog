1. 编辑value.yaml, 找到server.replicas: 1, 修改成replicas: 3
2. 编辑values.yaml: 找到：podAntiAffinity, 添加pod反亲和逻辑：**注意缩进**
      preferredDuringSchedulingIgnoredDuringExecution:
        - podAffinityTerm:
           labelSelector:
            matchLabels:
              app: consul
              component: server
              release: consul
           topologyKey: topology.kubernetes.io/zone
          weight: 100`