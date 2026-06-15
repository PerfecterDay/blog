# 双写问题及解决方案

在分布式系统架构中，**双写问题（Dual Write Problem）** 是一个经典且高频出现的数据一致性挑战。它指的是：当一次业务操作需要同时修改两个或多个独立系统的数据时，如何保证这些修改要么全部成功，要么全部失败，从而维持跨系统的数据一致性。

一个最典型的场景是：用户完成一笔订单支付后，系统既要更新数据库中的订单状态，又要向消息队列发送一条"支付成功"事件，供下游服务（如库存系统、通知系统、积分系统）消费。如果数据库更新成功但消息发送失败，下游服务将永远收不到通知；反之，如果消息发送成功但数据库更新失败，则会造成"虚假通知"。这两种情况都会导致严重的数据不一致。

本文将系统性地剖析双写问题的本质、难点，并深入探讨业界主流的解决方案，从应用层模式到基础设施层协议，帮助读者在实际项目中做出合理的技术选型。

---

## 一、问题本质与难点分析

### 1.1 什么是双写问题

双写问题并非仅指"写两次数据库"，而是泛指**任何需要在多个独立数据存储或服务之间进行协调写操作**的场景。这些存储或服务可能包括：

* 关系型数据库 + 消息队列（MQ）
* 关系型数据库 + 缓存（Redis）
* 主数据库 + 从数据库 / 读副本
* 不同业务域的微服务各自的数据库
* 数据库 + 搜索引擎（Elasticsearch）
* 数据库 + 外部第三方 API

以"数据库 + 消息队列"这一最经典的组合为例，用户操作涉及数据库变更和对外发送事件，如下图所示：

![dual-write.svg](/dual-write.svg)

**核心诉求**：这两个操作必须保证原子性——要么全做，要么全不做。

### 1.2 为什么简单的顺序执行不可行

许多开发者初次面对双写问题时，会直觉性地想到两种顺序策略：

**策略A：先发消息，再写数据库**

```
1. 发送 MQ 消息
2. 更新数据库
```

* **风险**：如果步骤1成功、步骤2失败，消息已发出但数据库未更新。下游消费者读到消息后去查数据库，发现数据不存在或状态不匹配，造成业务混乱。
* **补偿困难**：已发出的消息很难"撤回"。

**策略B：先写数据库，再发消息**

```
1. 更新数据库
2. 发送 MQ 消息
```

* **风险**：如果步骤1成功、步骤2失败（MQ宕机、网络中断、应用崩溃），数据库已更新但事件未发出。下游系统永远感知不到这次变更。
* **补偿困难**：应用已经崩溃或网络已断，无法自动重试发送。

> **Hugo 的踩坑记录**：早期项目中曾采用"先写库后发消息"的方案，并简单地用 try-catch 包裹消息发送逻辑，在 catch 块中记录错误日志。结果生产环境多次出现 MQ 瞬时闪断，虽然数据库更新成功，但消息丢失，下游库存系统未扣减，导致超卖。事后复盘发现，仅靠日志记录无法保证可靠投递，必须引入持久化机制。

### 1.3 真正的难点：故障场景下的保证

在数据库和消息系统都正常运行时，顺序执行两个操作确实可以都成功。问题的关键在于：**如何在网络中断、MQ宕机、系统断电等故障场景下，依然保证数据库与消息的一致性**。

配置 MQ 的消息持久化（Message Persistence）只能保证消息在 MQ 内部不丢失，但无法保证"消息一定被发送"这个行为本身。问题的本质是：**现有的事务实现（如数据库的 ACID 事务、MQ 的事务消息）通常只能作用于单个系统内部，跨系统的事务一致性必须由应用层自行处理**。

这正是双写问题的核心难点所在。

---

## 二、解决方案全景图

业界对双写问题的讨论非常广泛，从开源社区到商业化服务提供商都在探索解决方案：

* **Cockroach DB** 讨论了数据库与 Kafka 结合时的双写问题[[1]](#fn1)
* **RedHat 社区** 分享了在事件驱动应用中避免双写的实践经验[[2]](#fn2)
* **InfoQ** 探讨了 Saga 编排与 Outbox 模式的结合[[3]](#fn3)
* **RazorPay（印度支付公司）** 分享了分布式系统中实现可靠双写的工程实践[[4]](#fn4)

综合各方方案，处理双写问题的思路大体可分为以下几类：

| 方案类别 | 代表模式/技术 | 侵入性 | 复杂度 | 适用场景 |
| --- | --- | --- | --- | --- |
| **应用层模式** | Outbox 模式 | 中 | 低 | 微服务架构，数据库+MQ |
| **基础设施层** | CDC（Change Data Capture） | 低 | 中 | 已有数据库，需反向同步 |
| **混合方案** | Outbox + CDC | 低 | 中 | 追求低侵入与领域语义 |
| **协议层** | XA 分布式事务 | 低 | 高 | 强一致性要求，性能可接受 |
| **架构风格** | 事件溯源（Event Sourcing） | 高 | 高 | 全新系统设计，事件为核心 |
| **底层协议** | 强化 TCP 持久化 | 极低 | 极高 | 自研基础设施 |

> **关键洞察**：双写问题的解决方案——Outbox 模式，本质上就是**协同式 Saga（Choreography-based Saga）** 在两两服务协调中的基本模式。当涉及多个服务时，可以扩展为完整的 Saga 模式。更多 Saga 细节可参考 [Sagas 模式](/tech/patterns/sagas)。

---

## 三、Outbox 模式（发件箱模式）

### 3.1 核心思想

Outbox 模式的核心思想是：**利用数据库自身的事务能力，将"业务数据变更"和"待发送消息"绑定在同一个本地事务中**。具体做法是：

1. 在业务数据库中引入一张专门的 `outbox` 表，用于记录待发送的消息。
2. 业务操作在同一个数据库事务中完成：更新业务表 + 插入 outbox 记录。
3. 由另一个独立的任务（Poller / Relay）定期扫描 outbox 表，将待发送消息投递到消息系统。
4. 消息发送成功后，标记 outbox 记录为"已发送"或删除该记录。

如下图所示：

![outbox.svg](/outbox.svg)

### 3.2 为什么这样能保证一致性

关键在于**步骤1和步骤2在同一个本地事务中**：

```
BEGIN TRANSACTION;
  UPDATE orders SET status = 'PAID' WHERE id = 123;
  INSERT INTO outbox (topic, payload, created_at)
  VALUES ('order.paid', '{"orderId":123,"amount":99.99}', NOW());
COMMIT;
```

* 如果事务提交成功：业务数据已更新，outbox 记录也已写入，消息一定会被后续投递。
* 如果事务回滚：业务数据未更新，outbox 记录也未写入，消息不会被投递。
* 如果事务提交后、消息投递前系统崩溃：outbox 记录仍在数据库中，Poller 重启后会重新扫描并投递，保证**至少一次投递（At-Least-Once Delivery）**。

### 3.3 实现要点与最佳实践

**1. 消息投递的幂等性**

由于 Poller 可能重复投递同一条消息（例如投递后标记"已发送"前崩溃），消费者必须实现幂等性。常见做法：

* 在消息中携带唯一标识（如 `messageId`），消费者根据 `messageId` 去重。
* 利用数据库的唯一约束或分布式锁防止重复处理。

**2. Poller 的设计**

```
# 伪代码示例
while True:
    pending = db.query("SELECT * FROM outbox WHERE sent = false LIMIT 100")
    for msg in pending:
        try:
            mq.publish(msg.topic, msg.payload)
            db.execute("UPDATE outbox SET sent = true WHERE id = ?", msg.id)
        except Exception:
            # 发送失败，不标记 sent，下次重试
            log.error(f"Failed to send message {msg.id}")
    sleep(1)  # 控制轮询频率
```

**3. 消息顺序保证**

如果业务要求消息严格有序：

* 按 `created_at` 或自增 ID 排序扫描 outbox。
* 同一聚合根的消息由同一个 Poller 实例处理，避免并发导致的乱序。
* 或使用分区策略，确保同一业务键的消息进入同一分区。

**4. 性能优化**

* outbox 表应建立 `(sent, created_at)` 复合索引，加速待发送消息的查询。
* 已发送消息应及时清理（可设置 TTL 或归档策略），防止表无限膨胀。
* Poller 可采用批量读取 + 批量发送，减少网络往返。

### 3.4 优缺点分析

**优点**：

* 实现简单，不依赖外部中间件。
* 利用数据库本地事务，天然保证原子性。
* 已有成熟的开源实现可供直接集成[[5]](#fn5)。

**缺点**：

* **业务侵入性**：每个需要发消息的服务都需要维护 outbox 表和 Poller 逻辑。
* **设计复杂度**：在微服务架构下，让每个服务都带一个 outbox 表，会增加服务的设计复杂度。
* **延迟性**：消息不是实时发送的，取决于 Poller 的轮询间隔。

> **Hugo 的项目经验**：在支付中台项目中，我们要求所有涉及"状态变更+事件通知"的服务统一接入 Outbox 组件。通过封装一个 Spring Boot Starter，将 outbox 表操作和 Poller 逻辑下沉到公共库中，业务代码只需调用 `eventPublisher.publish(event)`，框架自动完成事务绑定和后续投递。这样将侵入性降到了最低，全团队 20+ 个服务在 2 周内全部完成改造。

更多关于 Outbox 模式的细节，可以参考 [Outbox 模式](/tech/patterns/outbox)。

---

## 四、CDC（Change Data Capture）方案

### 4.1 核心思想

CDC（变更数据捕获）本质上是对 Outbox 模式的**泛化实现**。它不侵入业务逻辑，而是通过监听数据库的变更日志（如 MySQL 的 binlog、PostgreSQL 的 WAL、MongoDB 的 oplog），自动捕获数据变更并转化为事件发送到消息系统。

如下图所示：

![cdc.svg](/cdc.svg)

### 4.2 CDC 的工作机制

以 PostgreSQL 的 Logical Replication 为例：

1. 数据库将 WAL（Write-Ahead Log）中的变更以逻辑格式输出。
2. CDC 连接器（如 Debezium）订阅这些逻辑变更流。
3. 连接器将变更事件转换为标准格式（如 JSON），发送到 Kafka 等消息系统。
4. 下游消费者从 Kafka 消费事件。

```
数据库写操作 → WAL/binlog → CDC 连接器 → Kafka → 下游消费者
```

### 4.3 主要问题与局限性

尽管 CDC 方案听起来很理想，但在实际应用中存在几个关键问题：

**1. 泄露实体结构**

CDC 采集的是存储层的数据变更，直接暴露内部数据表结构。虽然可以在发布消息前进行格式转换，但这会增加整体复杂度。

> **Hugo 的内部约定**：我们在使用 CDC 时，规定 CDC 发布的事件必须经过一层"领域事件转换器"，将原始表变更映射为业务语义化的领域事件。例如，将 `users` 表的 `UPDATE status='ACTIVE'` 转换为 `UserActivatedEvent`，包含用户ID、激活时间等业务字段，而非直接暴露表结构。

**2. 并非所有事件都有数据变更**

CDC 只能捕获数据存储层的变更，但：

* 某些业务事件可能不触发数据变更（如"用户浏览商品"可能只记录日志）。
* 某些数据变更可能并非由业务事件引起（如定时任务的数据清理）。

本质上，**领域事件是业务对象，而 CDC 采集的是存储层数据**。想要让 CDC 发布的事件真正符合领域模型，本质上是要做一次 ORM 的逆运算。

**3. 过度解耦的风险**

CDC 可以屏蔽下游对上游数据库的依赖，但并非所有依赖都应该被屏蔽。例如，在订单服务（Order）和支付服务（Payment）之间，这是一个**业务强依赖**：支付服务必须知道订单的存在和状态。用 CDC 这种极度松耦合的模式，可能会把本来应该显式声明的依赖强行消解掉，导致系统架构意图模糊。

**4. 数据库能力限制**

CDC 严重依赖于数据库本身的能力。以 PostgreSQL Logical Replication 为例[[6]](#fn6)：

| 限制项 | 说明 |
| --- | --- |
| 对象类型 | 只支持普通表，不支持序列、视图、物化视图、外部表、分区表和大对象 |
| 操作类型 | 只支持 DML（INSERT、UPDATE、DELETE），不支持 TRUNCATE、DDL |
| 表配置 | 需要同步的表必须设置 `REPLICA IDENTITY`，不能为 `NOTHING`（默认是 `DEFAULT`），且表中必须包含主键 |
| 版本兼容 | 发布端和订阅端的 PostgreSQL 主版本必须一致 |

### 4.4 已有开源实现

* **[Debezium](https://debezium.io/)**：最流行的开源 CDC 平台，支持 MySQL、PostgreSQL、MongoDB、SQL Server 等多种数据库。
* **Maxwell's Daemon**：专注于 MySQL binlog 解析。
* **Canal**：阿里巴巴开源的 MySQL binlog 增量订阅和消费组件。

---

## 五、结合 Outbox 与 CDC 的混合方案

### 5.1 设计动机

Outbox 模式的缺点和 CDC 的优点正好互补：

| Outbox 缺点 | CDC 优点 |
| --- | --- |
| 业务侵入性（需维护 outbox 表） | 不侵入业务逻辑 |
| 需编写 Poller 逻辑 | 自动捕获变更 |
| 领域语义清晰（业务定义事件） | 领域语义模糊（存储层事件） |

因此，不难得出一个**集合二者优点**的方案：

> **用 Outbox 存放对外的领域事件，然后利用 CDC 将 Outbox 中的数据发送到消息系统中。**

### 5.2 架构设计

这样设计的优势在于：

1. **领域语义清晰**：使用方只需定义领域事件的结构（放入 outbox），事件内容完全由业务控制。
2. **避免暴露内部模型**：不对外暴露内部数据对象的存储模型。
3. **无需编写 Poller**：CDC 自动捕获 outbox 表的变更并发送到消息系统，不必编写额外的轮询逻辑。

其基本设计如下图所示：

![outbox_with_cdc.svg](/outbox_with_cdc.svg)

### 5.3 工作流程

```
1. 业务事务：UPDATE 业务表 + INSERT outbox 表（领域事件）
2. 数据库 WAL/binlog 记录 outbox 插入
3. CDC 连接器捕获 outbox 变更
4. 连接器将事件发送到 Kafka
5. 下游消费者消费领域事件
```

关于这个实现方案的更多细节，可以参考[这篇文章](https://medium.com/codex/outbox-pattern-for-reliable-data-exchange-between-microservices-9c938e8158d9)。

### 5.4 适用场景

* 已有数据库基础设施，希望引入事件驱动架构。
* 团队希望保持领域事件的语义清晰，同时减少应用层代码。
* 使用 PostgreSQL / MySQL 等支持 CDC 的数据库。

---

## 六、支持事务的消息系统（XA 分布式事务）

### 6.1 核心思想

如果消息中间件把自己模拟成数据库，并支持数据库的 **XA 分布式事务协议**，便可以让消息发送与数据库变更在同一个分布式事务中完成。XA 协议由 X/Open 组织提出，定义了分布式事务处理的规范，通过两阶段提交（2PC）保证跨多个资源管理器的事务原子性。

### 6.2 工作流程

```
1. 开启全局事务（Transaction Manager）
2. 注册数据库资源（XA Resource 1）
3. 注册 MQ 资源（XA Resource 2）
4. 执行业务 SQL（数据库分支事务）
5. 发送 MQ 消息（MQ 分支事务）
6. 两阶段提交：
   - Phase 1（Prepare）：询问所有资源是否可提交
   - Phase 2（Commit）：所有资源确认后统一提交
```

### 6.3 已知支持 XA 的消息中间件

| 中间件 | XA 支持情况 | 备注 |
| --- | --- | --- |
| **RocketMQ** | ✅ 支持事务消息 | 通过半消息 + 回查机制实现[[7]]|
| **endurox** | ✅ 支持 XA | 开源事务处理框架 |
| RabbitMQ | ❌ 不支持 | 官方明确不推荐在 RabbitMQ 中使用 XA[[8]]|
| ActiveMQ | ❌ 不推荐 | 性能影响严重 |
| Kafka | ❌ 不支持 | Kafka 设计哲学与 XA 不兼容 |

### 6.4 为什么不普遍使用

更常见的消息中间件（RabbitMQ、ActiveMQ、Kafka）均不支持或不推荐 XA 事务。原因也很简单：**影响性能**[[8:1]](#fn8)。

XA 两阶段提交的网络往返和协调开销，在高并发场景下会成为严重瓶颈。此外，2PC 还存在协调者单点故障、阻塞等问题。因此，XA 分布式事务通常只在**强一致性要求且并发量可控**的场景中使用。

> **Hugo 的技术选型建议**：在支付核心链路中，如果确实需要强一致性，优先考虑 RocketMQ 的事务消息（基于半消息机制，性能优于纯 XA）。对于一般业务场景，Outbox 模式的性能和复杂度平衡更好。

---

## 七、事件溯源（Event Sourcing）

### 7.1 核心思想

Martin Fowler 在《Event Sourcing》一文中这样阐述[[9]](#fn9)：

> The official system of record can either be the event logs or the current application state. If the current application state is held in a database, then the event logs may only be there for audit and special processing. Alternatively the event logs can be the official record and databases can be built from them whenever needed.

事件溯源的核心是**将事件本身作为系统的中心**，而非数据库中的当前状态。所有状态变更都通过追加事件来实现，系统的当前状态可以通过重放所有事件来重建。

### 7.2 架构设计

如果不以数据库为系统的中心，而将事件本身作为数据的中心，架构如下：

![event-sourcing.svg](/event-sourcing.svg)

**工作流程**：

1. 业务操作产生领域事件，直接追加到事件存储（Event Store）。
2. 事件存储同时作为消息系统，将事件分发给感兴趣的消费者。
3. 消费者根据事件更新各自的读模型（Read Model / Projection）。
4. 如果需要查询当前状态，可以通过重放事件或查询物化视图。

如果对于"事件自产自消"的行为有顾虑（即同一个服务既生产事件又消费事件来更新自己的数据库），也可以把消费事件持久化到数据库的过程放在独立的节点来执行：

![event-sourcing-2.svg](/event-sourcing.svg)

### 7.3 事件溯源 vs CDC

Debezium 官方博客有一篇很好的对比文章[[10]](#fn10)：

| 维度 | Event Sourcing | CDC |
| --- | --- | --- |
| 事件来源 | 应用层主动生成领域事件 | 数据库层被动捕获数据变更 |
| 事件语义 | 业务语义，表达"发生了什么" | 技术语义，表达"数据怎么变了" |
| 事件存储 | 事件是唯一的真相来源 | 数据库是真相来源，事件是派生 |
| 应用侵入性 | 高（需重构为事件驱动） | 低 |
| 适用场景 | 全新系统设计 | 已有系统改造 |

### 7.4 为什么不只为解决双写而引入

> ![🔔](/_assets/svg/twemoji/1f514.svg) **事件溯源作为一种系统架构风格，是关系整个系统设计的重大决策，一般并不会为"解决双写问题"而引入事件溯源。** 这里只是为了解决方案的完整性将其列出。在实际项目中，需要有更好的理由来引入事件溯源，例如：
>
> * 需要完整的审计追踪（Audit Trail）
> * 需要支持时间旅行查询（Temporal Query）
> * 需要多维度读模型（CQRS 模式）
> * 系统天然适合事件驱动（如交易撮合、游戏状态同步）

---

## 八、其它方案与前沿探索

### 8.1 底层协议强化

以上方案都是应用层方案。应用层方案只能基于现有技术栈的能力来解决问题。但双写问题之所以产生，有两种主要原因：

1. **服务自身出错**：应用逻辑缺陷、Bug 等。
2. **网络及基础设施故障**：MQ 宕机、网络中断、系统断电等——这也是更主要的原因。

如果能通过**自定义网卡驱动**对 TCP 协议本身进行强化，让连接本身做持久化，断掉之后能自动重连并且自动续传，然后所有应用层再基于这个加强版 TCP 协议去做应用层协议，也可以避免双写问题产生。

**但是**：这种做法的成本极高，需要从底层协议栈到所有中间件应用全部自研才能实现，没有现成的开源实现可以参考。目前仅在极少数超大规模基础设施团队中有类似探索（如 Google 的 TCP BBR、自定义 RDMA 协议等）。

### 8.2 现代云原生方案

随着云原生技术的发展，一些新的思路正在涌现：

**1. Dapr 的 Pub/Sub 组件**

[Dapr](https://dapr.io/)（Distributed Application Runtime）提供了统一的消息发布抽象，其 Pub/Sub 组件可以与多种 MQ 后端集成，并通过 Sidecar 模式简化应用代码。

**2. 云数据库的内置 CDC**

AWS Aurora、Azure Cosmos DB、阿里云 PolarDB 等云数据库开始内置 CDC 能力，用户无需部署额外的 Debezium 等组件，直接在控制台配置即可将变更流式输出到消息服务。

**3. Serverless 事件架构**

AWS EventBridge、Azure Event Grid 等 Serverless 事件总线服务，提供了托管的事件路由和转换能力，可以与数据库触发器、API 网关等集成，降低自建事件基础设施的成本。

---

## 九、方案选型决策树

面对双写问题，如何选择合适的方案？以下决策树可供参考：

```
是否需要强一致性（金融级）？
├── 是 → 并发量是否可控？
│   ├── 是 → RocketMQ 事务消息 / XA 分布式事务
│   └── 否 → Outbox 模式 + 幂等消费者
│
└── 否 → 是否已有成熟数据库基础设施？
    ├── 是 → 是否希望低侵入？
    │   ├── 是 → CDC（Debezium）
    │   └── 否 → Outbox 模式
    │
    └── 否 → 是否全新系统设计？
        ├── 是 → 事件溯源（Event Sourcing）
        └── 否 → Outbox 模式（最通用）
```

### 选型建议总结

| 场景 | 推荐方案 | 理由 |
| --- | --- | --- |
| 一般微服务（数据库+MQ） | **Outbox 模式** | 简单、可靠、通用性强 |
| 已有数据库，需反向同步 | **CDC** | 不侵入业务，自动捕获 |
| 追求低侵入+领域语义 | **Outbox + CDC** | 二者优点结合 |
| 支付/金融核心链路 | **RocketMQ 事务消息** | 强一致性，性能可接受 |
| 全新系统设计，事件为核心 | **事件溯源** | 架构层面统一，但成本高 |
| 超大规模，自研基础设施 | **底层协议强化** | 终极方案，但投入巨大 |

---

## 十、Hugo 的工程实践总结

### 10.1 内部约定：双写问题的处理原则

在我们团队的工程规范中，对双写问题的处理有以下约定：

1. **默认使用 Outbox 模式**：所有涉及"状态变更+事件通知"的场景，优先使用 Outbox。
2. **统一封装公共组件**：通过公司内部的消息发布 SDK，将 outbox 表操作、Poller 逻辑、幂等控制等全部下沉，业务代码无感知。
3. **CDC 作为补充**：仅在已有系统改造、无法修改业务代码的场景下使用 CDC，且必须经过领域事件转换层。
4. **禁止裸用"先写库后发消息"**：任何直接在生产代码中先更新数据库再调用 MQ 发送的做法，在 Code Review 中必须被打回。
5. **消费者必须幂等**：所有事件消费者必须实现幂等性，这是使用 Outbox / CDC 的前提条件。

### 10.2 踩坑记录

**坑1：Outbox 表无限膨胀**

早期实现中未清理已发送的 outbox 记录，导致表在 3 个月内增长到数千万行，查询性能急剧下降。

**解决方案**：

* 已发送记录迁移到归档表（按月分表）。
* 主表只保留 7 天内的未发送记录和 1 天内的已发送记录。
* 建立 `(sent, created_at)` 复合索引。

**坑2：CDC 事件顺序错乱**

使用 Debezium 捕获 PostgreSQL 变更时，由于多个表并发变更，Kafka 中的事件顺序与业务发生顺序不一致，导致下游状态机处理异常。

**解决方案**：

* 对同一聚合根的事件使用相同的 Kafka Partition Key，确保单分区有序。
* 在事件中加入 `causationId` 和 `correlationId`，下游可通过因果链重建顺序。

**坑3：消息格式演进**

Outbox 中存储的消息格式在业务迭代中发生变更，但旧格式消息仍在 Poller 的待发送队列中，导致序列化失败。

**解决方案**：

* 消息格式引入版本号（`schemaVersion`）。
* Poller 根据版本号选择对应的序列化器。
* 重大格式变更时，先升级消费者，再升级生产者。

---

## 十一、总结

双写问题是分布式系统中数据一致性的经典挑战。本文系统性地梳理了从应用层到基础设施层、从协议层到架构风格的多种解决方案：

* **Outbox 模式**：最通用、最推荐的方案，利用数据库本地事务保证原子性。
* **CDC**：低侵入的反向同步方案，适合已有系统改造。
* **Outbox + CDC**：二者优点结合，兼顾领域语义和自动化。
* **XA 分布式事务**：强一致性方案，但性能代价高。
* **事件溯源**：架构层面的终极方案，但改造成本极高。
* **底层协议强化**：理论可行，但工程投入巨大。

在实际项目中，**Outbox 模式**通常是最佳起点——它简单、可靠、不依赖特定中间件，且已有成熟的开源实现。随着系统演进，可以逐步引入 CDC 或考虑事件溯源等更高级的架构风格。

关键在于：**没有银弹**。选择方案时需要综合考虑一致性要求、性能需求、团队能力、现有基础设施和改造成本。

---

## 引用
1. [Message Queuing and the Database: Solving the Dual Write Problem](https://www.cockroachlabs.com/blog/message-queuing-database-kafka/) — Cockroach Labs [↩︎](#fnref1)
2. [Avoiding dual writes in event-driven applications](https://developers.redhat.com/articles/2021/07/30/avoiding-dual-writes-event-driven-applications) — RedHat Developers [↩︎](#fnref2)
3. [Saga Orchestration for Microservices Using the Outbox Pattern](https://www.infoq.com/articles/saga-orchestration-outbox/) — InfoQ [↩︎](#fnref3)
4. [Achieving reliable dual writes in distributed systems](https://engineering.razorpay.com/achieving-reliable-dual-writes-in-distributed-systems-cb9ff3b9bfc1) — RazorPay Engineering [↩︎](#fnref4)
5. [Outbox 模式开源实现列表](https://www.libhunt.com/search?query=outbox) [↩︎](#fnref5)
6. [PostgreSQL Logical Replication](https://developer.aliyun.com/article/585446) — 阿里云开发者社区 [↩︎](#fnref6)
7. [RocketMQ 事务消息](https://github.com/apache/rocketmq/blob/develop/docs/cn/features.md#7-%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF) — Apache RocketMQ 官方文档 [↩︎](#fnref7)
8. [Should I use XA](https://activemq.apache.org/should-i-use-xa) — Apache ActiveMQ 官方文档 [↩︎][↩︎](#fnref8:1)
9. [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) — Martin Fowler [↩︎](#fnref9)
10. [Event Sourcing vs CDC](https://debezium.io/blog/2020/02/10/event-sourcing-vs-cdc/) — Debezium 官方博客 [↩︎](#fnref10)

[沪ICP备15048960号-1](https://beian.miit.gov.cn/)
