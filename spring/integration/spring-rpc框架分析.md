## multi-protocol-framework-springboot
SpringBootAutoConfiguration 中注入了以下 bean:
+ SgPlaceHolder
+ MultiSgSyncClientProcessor ： 处理 `@MultiSgSyncClient` 注解
+ SgSpringListener ：主要是监听 spring 的 `ContextRefreshedEvent` 和 `ContextClosedEvent` 事件
+ SgSpringConfig ： 当使用 XML 配置时生效
+ ConfigurationContainer ： 使用 yaml 配置时生效
+ SgMultiLifecycle (SgExternalMultiLifecycle) ：没有使用 spring 的 LifeCycle ，


### SgSpringListener
SgSpringListener 监听 ContextRefreshedEvent/ContextClosedEvent：
+ ContextRefreshedEvent： 开启一个异步线程调用 `lifecycle.startup();`
+ ContextClosedEvent：调用 `lifecycle.close();`

### SgMultiLifecycle

SgMultiLifecycle.startup

### MultiSgSyncClientProcessor 处理 `MultiSgSyncClient` 注解
MultiSgSyncClientProcessor 继承自 SmartInstantiationAwareBeanPostProcessor
    SynchronousConsumerManager.getConsumerInstance(injectedType, builder.build()); 生成bean
        Object object = Proxy.newProxyInstance(clazz.getClassLoader(),
                new Class<?>[]{clazz}, new SynchronousConsumerInvocationHandler<>(clazz, rpcOptions)); 使用JDK动态代理生成代理对象

                 ClientInvokeTargetFilter->SgMultiHttpConnection 发起实际调用
                 Object proxy = Forest.config(providerNode.getIdentifier()).createInstance(clazzName); 使用 Forest 发起远程调用
                 return methodName.invoke(proxy, args);


### 服务调用动态代理 `SynchronousConsumerInvocationHandler`
pre-filters:
`CustomerFullFillInfoFilter`
`CustomerTraceKeyFullFillFilter` 
`ConsumerPreSimpleTrace`
`CustomerRequestLogFilter`
`CustomerMergerSearchFilter` 服务查询
`CustomerConfigSearchFilter`
`CustomerAddAuthFilter`
`ConsumerFaultInjectFilter`


around-filters:
`ClientFailOverFilter`
`ConsumerSimpleControlFilter`
`ProviderNodeFilter` 获取节点
`ChooseNodeFilter`
`ConsumerAroundSimpleTrace`
`ConnectionFetchFilter` 获取 RPC connection
`ClientInvokeTargetFilter`
`GrpcAsyncConsumerInvokerFilter`

post-filters:
`ConsumerPostOpenTracingTrace`
`CustomerResponseLogFilter`
`CustomerContextCleaningFilter`


配置类： `ConfigurationContainer`

## 注册中心相关-服务注册与解注册
在 `SgMultiLifecycle` 的 `startupRegistries` 方法中有一下代码段：
```
for (SgRegistry registry : registries) {
    //这里是异步的
    registry.initEnv();
}
```
会启动服务注册订阅功能。

`SgNacosRegistry`

namespace 命名规则是：配置中的 protocol + publish-type。这块的具体代码在 `SgNacosRegistry` 的 `startUp` 方法中创建 `namingService` 实例时。

注册参数：
```
app:unknown
metadata:{"side":"provider","appVersion":"1.0.0","pid":"98390","proxyHost":"","timeout":"5000","timeStamp":"1739861131989","retryTimes":"0","proxyPort":"","protocol":"http","application":"testChain","framework.version":"3.0.0-SNAPSHOT","internalVersion":"3.0.0.7","tag":"","lang":"java","dcCity":"SH","useTLS":"false","dc":"CS"}
ip:10.176.115.168
weight:100.0
ephemeral:true
serviceName:DEFAULT_GROUP@@testChain
accessToken:eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3OTEzMn0.gfKa3hI0o1SSWuRxqY5kbz1H28AvCahTqFDVQMddUo8
groupName:DEFAULT_GROUP
namespaceId:http-application
port:8099
enable:true
healthy:true
clusterName:
```

### nacos 服务监听监听
`SgNacosRegistry` 的 startup 方法中会监听服务下的实例列表变化。
```
namingService.subscribe(convertKeyToNacos(application),new ApplicationEventListener());
```

## multi-protocol-framework-http-starter
`HttpProviderInvokeFilter` 拦截 http 的调用

## SPIContainer
`SPIContainer` 自定义的 SPI，会扫描 jar 包中的 `META-INF/demo-multi-sg-services/` 路径下注册的服务

`getSPIImplementClasses` 读取SPI 文件，解析并保存名字与服务的映射，比如 `sg-registry-nacos` jar 中的SPI文件：
```
nacos-byService=com.demo.sg.multi.registry.nacos.SgNacosRegistry
nacos-byApplication=com.demo.sg.multi.registry.nacos.SgNacosRegistry
```

`sg-registry-zookeeper` 中的如下：
```
zookeeper-byService=com.demo.sg.multi.registry.zookeeper.SgZookeeperServiceRegistry
zookeeper-byApplication=com.demo.sg.multi.registry.zookeeper.SgZookeeperServiceRegistry
```

`sg-registry-local` 中的如下：
```
local-byService=com.demo.sg.multi.registry.local.SgLocalRegistry
local-byApplication=com.demo.sg.multi.registry.local.SgLocalRegistry
```

然后根据配置中的注册中心的类型和byApplication/byService来决定实例化的类型。


## 探活
SgMultilHttpExtConnectionPool -> orgValidate

sg-monitor-api 注册了 META-INF/demo-multi-sg-services/com.demo.sg.multi.monitor.SgMonitorBehavior SPI, `basic=com.demo.sg.multi.monitor.BasicSgMonitorBehavior`


HardWareUtil 获取主机IP  -DgtjaSgHost=xxx 指定绑定的 IP


## proto文件发布编译流程
proto 文件发布编译流程：
1. 登录全链接 OA -> 服务治理
2. 点击微服务管理下的 微服务发布菜单
3. 左侧找到香港国际业务平台->国际etradePro柜台对接服务系统
4. 新增应用或者选择已有的应用
5. 点击发布编译，上传包含 proto 文件的文件夹
6. 点击编译
7. 点击发布填写 GAV 信息
8. 查看信息 复制对应的 GAV 信息后添加依赖即可使用
