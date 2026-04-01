# nacos 注册中心API 

## 初始化SDK 
```
Properties properties = new Properties();
# 指定Nacos-Server的地址
properties.setProperty(PropertyKeyConst.SERVER_ADDR, "localhost:8848");
# 指定Nacos-SDK的命名空间
properties.setProperty(PropertyKeyConst.NAMESPACE, "${namespaceId}");

# 初始化配置中心的Nacos Java SDK
ConfigService configService = NacosFactory.createConfigService(properties);

# 初始化服务注册中心的Nacos Java SDK
NamingService namingService = NacosFactory.createNamingService(properties);
```





## 服务发现的API

### 注册实例
```
void registerInstance(String serviceName, String ip, int port) throws NacosException;
void registerInstance(String serviceName, String groupName, String ip, int port) throws NacosException;
void registerInstance(String serviceName, String ip, int port, String clusterName) throws NacosException;
void registerInstance(String serviceName, String groupName, String ip, int port, String clusterName) throws NacosException;
void registerInstance(String serviceName, Instance instance) throws NacosException;
void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException;
```

### 注销实例
```
void deregisterInstance(String serviceName, String ip, int port) throws NacosException;
void deregisterInstance(String serviceName, String groupName, String ip, int port) throws NacosException;
void deregisterInstance(String serviceName, String ip, int port, String clusterName) throws NacosException;
void deregisterInstance(String serviceName, String groupName, String ip, int port, String clusterName) throws NacosException;
void deregisterInstance(String serviceName, Instance instance) throws NacosException;
void deregisterInstance(String serviceName, String groupName, Instance instance);
```

### 获取全部实例
```
List<Instance> getAllInstances(String serviceName) throws NacosException;
List<Instance> getAllInstances(String serviceName, String groupName) throws NacosException;
List<Instance> getAllInstances(String serviceName, boolean subscribe) throws NacosException;
List<Instance> getAllInstances(String serviceName, String groupName, boolean subscribe) throws NacosException;
List<Instance> getAllInstances(String serviceName, List<String> clusters) throws NacosException;
List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters) throws NacosException;
List<Instance> getAllInstances(String serviceName, List<String> clusters, boolean subscribe) throws NacosException;
List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters, boolean subscribe) throws NacosException;
```

### 获取健康或不健康实例列表
```
List<Instance> selectInstances(String serviceName, boolean healthy) throws NacosException;
List<Instance> selectInstances(String serviceName, String groupName, boolean healthy) throws NacosException;
List<Instance> selectInstances(String serviceName, boolean healthy, boolean subscribe) throws NacosException;
List<Instance> selectInstances(String serviceName, String groupName, boolean healthy, boolean subscribe) throws NacosException;
List<Instance> selectInstances(String serviceName, List<String> clusters, boolean healthy) throws NacosException;
List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy) throws NacosException;
List<Instance> selectInstances(String serviceName, List<String> clusters, boolean healthy, boolean subscribe) throws NacosException;
List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy, boolean subscribe) throws NacosException;
```

### 监听服务
```
void subscribe(String serviceName, EventListener listener) throws NacosException;
void subscribe(String serviceName, String groupName, EventListener listener) throws NacosException;
void subscribe(String serviceName, List<String> clusters, EventListener listener) throws NacosException;
void subscribe(String serviceName, String groupName, List<String> clusters, EventListener listener) throws NacosException;
```

### 批量注册/注销服务实例
由于同一个Nacos Client实例，仅能向一个服务注册一个实例；若同一个Nacos Client实例多次向同一个服务注册实例，后注册的实例将会覆盖先注册的实例。 考虑到社区存在代理注册的场景：如Nacos-Sync， Proxy-Registry等，需要在一个客户端中注册同一个服务的不同实例，社区新增了批量注册服务实例的功能。
```
void batchRegisterInstance(String serviceName, String groupName, List<Instance> instances) throws NacosException;
```

批量注销服务：
```
void batchDeregisterInstance(String serviceName, String groupName, List<Instance> instances) throws NacosException;
```