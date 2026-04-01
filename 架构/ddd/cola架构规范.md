# COLA 架构规范
{docsify-updated}

> https://modelbaba.com/architecture/ddd/2100.html   
> https://www.baeldung.com/hexagonal-architecture-ddd-spring

[Github链接](https://github.com/alibaba/COLA)

### 分层架构
<center><img src="pics/cola.png" width="54%">
<img src="pics/cola-layer.png" width="45%"></center>


+ User Interface 为用户界面层，向用户展示信息和传入用户命令。这里指的用户不单单只使用用户界面的人，也可能是外部系统，诸如用例中的参与者。
+ Application 为应用层，用来协调应用的活动，不包含业务逻辑，通过编排领域模型，包括领域对象及领域服务，使它们互相协作。不保留业务对象的状态，但它保有应用任务的进度状态。
+ Domain 为领域层，负责表达业务概念，业务状态信息以及业务规则。尽管保存业务状态的技术细节是由基础设施层实现的，但是反映业务情况的状态是由本层控制并且使用的。领域层是业务软件的核心，领域模型位于这一层。
+ Infrastructure 为基础实施层，提供公共的基础设施组件，如持久化机制、消息管道的读取写入、文件服务的读取写入、调用邮件服务、对外部系统的调用等等。

<center><img src="pics/cola-ac.png" width="60%"></center>

### POJO 定义规范
+ `DO（Data Object)`：此对象与数据库表结构一一对应，通过 DAO层向上传输数据源对象。
+ `DTO（Data Transfer Object）`：数据传输对象，Service或 Manager向外传输的对象。
+ `BO（Business Object）`：业务对象，由 Service层输出的封装业务逻辑的对象。
+ `AO（Application Object）`： 应用对象， 在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高。
+ `VO（View Object）`：显示层对象，通常是 Web向模板渲染引擎层传输的对象。
+ `Query`：数据查询对象，各层接收上层的查询请求。注意超过2个参数的查询封装，禁止使用Map类来传输。

```
Controller
    ->接收DTO
    ->调用Service
Service
    ->DTO->BO
    ->业务处理（核心逻辑）
    ->BO->Entity
    ->持久化
    ->返回BO→VO
Repository
    ->操作Entity→DB
```


### 定义
1. 实体（ENTITY）  
	实体有唯一的标识，有生命周期且具有延续性。例如一个交易订单，从创建订单我们会给他一个订单编号并且是唯一的这就是实体唯一标识。同时订单实体会从创建，支付，发货等过程最终走到终态这就是实体的生命周期。订单实体在这个过程中属性发生了变化，但订单还是那个订单，不会因为属性的变化而变化，这就是实体的延续性。

2. 值对象（VALUE-OBJECT）  
	通过对象属性值来识别的对象，它将多个相关属性组合为一个概念整体。在 DDD 中用来描述领域的特定方面，并且是一个没有标识符的对象，叫作值对象。值对象没有唯一标识，没有生命周期，不可修改，当值对象发生改变时只能替换（例如String的实现）。
3.  


### 组件
组件名称 | 功能 | 依赖
------ | ---- | ----
`cola-component-dto` | 定义了`DTO`格式，包括分页 |无
`cola-component-exception` | 定义了异常格式，<br>主要有`BizException`和`SysException` |无
`cola-component-statemachine` | 状态机组件 | 无
`cola-component-domain-starter` | `Spring`托管的领域实体组件 | 无
`cola-component-catchlog-starter` | 异常处理和日志组件 | `exception`、`dto`组件
`cola-component-extension-starter` | 扩展点组件 | 无
`cola-component-test-container` | 测试容器组件 | 无

### archetype
1. Web
```
mvn archetype:generate -DgroupId='com.panda.baicy' -DartifactId='demo-web' -Dversion='1.0.0-SNAPSHOT' -Dpackage='com.panda.baicy' -DarchetypeArtifactId='cola-framework-archetype-web' -DarchetypeGroupId='com.alibaba.cola' -DarchetypeVersion='4.3.2'
```
2. Service
```
mvn archetype:generate \
    -DgroupId=com.alibaba.cola.demo.service \
    -DartifactId=demo-service \
    -Dversion=1.0.0-SNAPSHOT \
    -Dpackage=com.alibaba.demo \
    -DarchetypeArtifactId=cola-framework-archetype-service \
    -DarchetypeGroupId=com.alibaba.cola \
    -DarchetypeVersion=4.3.2
```

### 依赖关系
start -> adapter -> app -> client, infrastructure -> domain
<center>
<img src="pics/cola-arch.png" width="50%">
<img src="pics/cola-pom.png" width="48%">
</center>