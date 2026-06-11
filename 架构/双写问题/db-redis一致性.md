# 高并发下数据库与 Redis 缓存的一致性

{docsify-updated}

先说结论：**在分布式系统中无法做到强一致，只能在"最终一致 + 不一致窗口足够小 + 关键路径可兜底"之间做工程权衡。** 下面按"为什么难 → 主流方案 → 高并发下的坑 → 终极方案 → 选型决策"展开。

---

## 一、为什么"双写一致"本质上做不到强一致

操作两个独立的存储系统（DB + Redis）不在同一个事务里，任意时刻都可能：

- DB 成功，Redis 失败
- Redis 成功，DB 失败
- 两者都成功但**顺序交叉**导致旧值覆盖新值

这是典型的**双写问题（Dual Write Problem）**，与 [Outbox 模式](/架构/双写问题/outbox.md) 想解决的问题同源。

---

## 二、四种主流方案及其问题

### 2.1 先更新 DB，再更新缓存 ❌

```
线程A: 更新 DB(v=1) → 更新 Redis(v=1)
线程B: 更新 DB(v=2) → 更新 Redis(v=2)
```

并发交叉时：

```
A: 写DB=1 ──────────────→ 写Redis=1
B:        写DB=2 → 写Redis=2
```

结果可能是：DB=2，Redis=1 → **脏数据长期存在**。**不推荐**。

### 2.2 先更新 DB，再删除缓存（Cache Aside，最常用）✅

```python
def update(id, value):
    db.update(id, value)
    redis.delete(key(id))   # 让下次读触发回源
```

读路径：

```python
def get(id):
    val = redis.get(key(id))
    if val is None:
        val = db.get(id)
        redis.setex(key(id), TTL, val)
    return val
```

**仍有极端竞态**：

```
T1: 读 Redis miss
T1: 读 DB 得到 v=旧
                           T2: 写 DB v=新
                           T2: 删 Redis（其实没东西可删）
T1: 写 Redis v=旧   ← 旧值被回填进缓存
```

T1 在 T2 之前读 DB，但在 T2 之后写 Redis，导致**缓存里是旧值**。

- **发生概率低**：要求"读回源 + 数据库写 + 删缓存"三步穿插在"读 DB 到写 Redis"之间
- 通常用 **短 TTL** 兜底（比如 5 分钟）即可接受

### 2.3 先删缓存，再更新 DB ❌

```
T1: 删 Redis
                   T2: 读 Redis miss → 读 DB（旧值）→ 写 Redis(旧)
T1: 写 DB(新)
```

→ Redis 留下旧值，且 DB 已经更新，**不一致窗口可以很长**。**更不推荐**。

### 2.4 延迟双删（兼顾方案 2/3 的尝试）⚠️

```python
def update(id, value):
    redis.delete(key(id))           # 第一次删
    db.update(id, value)
    sleep(500ms)                    # 等读流量回填完
    redis.delete(key(id))           # 第二次删
```

- 第二次删的目的：清掉"在我写 DB 期间被其他线程读 DB 旧值后回填进 Redis"的脏数据
- **缺点**：sleep 时间难定，太短没用，太长拖响应时间；线程阻塞浪费资源
- 改进：异步延迟双删（写消息到 MQ 延迟队列，几百毫秒后再发一次删除）

---

## 三、高并发下额外要处理的问题

### 3.1 缓存穿透（Cache Penetration）

查不存在的 key，每次都打到 DB。

- **空值缓存**：`SET key "NULL" EX 60`
- **布隆过滤器**：所有可能的 key 预先入 BloomFilter，不在则直接返回 null

### 3.2 缓存击穿（Hot Key Expire）

热点 key 突然过期，瞬间大量请求打到 DB。

- **互斥锁回源**（singleflight）：

```python
def get(id):
    val = redis.get(key(id))
    if val: return val
    if redis.set(lock_key(id), "1", nx=True, ex=10):  # 抢锁
        try:
            val = db.get(id)
            redis.setex(key(id), TTL, val)
        finally:
            redis.delete(lock_key(id))
    else:
        sleep(50ms); return get(id)   # 等待+重试
    return val
```

- **逻辑过期**：value 里塞个 `expire_at` 字段，物理上不过期，过期后由"第一个发现的请求"异步刷新，其他请求继续返回旧值

### 3.3 缓存雪崩（Mass Expire）

大量 key 同时过期 → DB 雪崩。

- TTL 加随机抖动：`TTL = base + random(0, 300s)`
- 多级缓存（本地 Caffeine + Redis），分散过期时点

### 3.4 删除失败 / 更新失败

网络抖动导致 `redis.del` 失败 → 长期不一致。**必须有补偿机制**（见下一节）。


---

## 四、生产级的"终极方案"：基于 binlog 的订阅式失效

这是大型互联网公司广泛采用的架构，本质上是**让 DB 成为唯一的真相源，缓存被动跟随**。

```
       ┌──────────┐
应用 ──→│ 数据库   │
       └──┬───────┘
          │ binlog
          ▼
   ┌────────────────┐
   │ Canal/Debezium │  订阅 binlog
   └──────┬─────────┘
          │
          ▼
   ┌───────────────┐
   │   消息队列    │  Kafka / RocketMQ
   └──────┬────────┘
          │
          ▼
   ┌───────────────┐
   │ 缓存更新服务  │  → 删除/更新 Redis
   └───────────────┘
```

**优势**：

1. 业务代码**只操作 DB**，不直接操作缓存 → 业务侧零侵入
2. binlog 顺序保证 → 不会出现"旧值覆盖新值"
3. 消息队列保证**至少一次** → 失败可重试
4. 端到端延迟可压到 **100ms 量级**

**代价**：需要运维 Canal/Debezium + Kafka，引入新组件。

**典型组合**：

- MySQL → Canal → RocketMQ → 缓存服务（阿里系常用）
- MySQL/PG/SQL Server → Debezium → Kafka → 缓存服务（国际主流）

---

## 五、订阅式 + Cache Aside 混合（最实用）

业务侧仍用 Cache Aside（保证常规路径低延迟），同时由 binlog 订阅做兜底：

```
写请求：
  应用 → 写 DB → 删 Redis（best effort）
  binlog → 异步再删一次 Redis（兜底）

读请求：
  应用 → 读 Redis → miss 时回源 DB
```

- 业务路径快
- 即使应用删缓存失败，binlog 链路也会兜底删一次
- 这是**目前绝大多数高并发系统的实际选择**

---

## 六、强一致性场景怎么办？

少数业务确实不能容忍任何不一致（账户余额、库存扣减），此时：

### 6.1 放弃缓存

直接走 DB，靠 DB 自身性能 + 读写分离 + 分库分表撑住流量。

### 6.2 分布式锁 + 双写

代价大、性能差，仅用于极少数关键写。

### 6.3 用 Redis 作为唯一真相源

比如库存预扣：所有读写都走 Redis，定时落 DB。Redis 持久化（AOF + 集群）保证不丢。

- 适合：库存、限流、计数、排行榜
- 不适合：要复杂查询和事务的业务数据

### 6.4 让缓存成为"物化视图"

不用 Cache Aside，缓存内容由 DB 通过 CDC 全量构建。读永远读缓存，写永远写 DB，**两者通过 CDC 单向同步**。配合版本号防止旧覆盖新。

---

## 七、并发写覆盖的防护：版本号 / CAS

无论用哪种方案，多写并发都可能让旧值覆盖新值。在缓存里加版本号：

```lua
-- Lua 脚本保证原子比较
local cur = redis.call('GET', KEYS[1])
if cur == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
end
local cur_ver = tonumber(string.match(cur, '"v":(%d+)'))
if cur_ver < tonumber(ARGV[2]) then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
end
return 0
```

版本号通常用 DB 的更新时间戳或自增 `version` 列。

---

## 八、选型决策树

```
是否能容忍秒级不一致？
├─ 是 → Cache Aside（短 TTL 兜底）            ← 90% 业务用这个
│   └─ 想更可靠 → 叠加 binlog 订阅兜底
│
└─ 否 → 是否高并发？
    ├─ 是 → binlog 订阅式（Canal/Debezium）     ← 大厂主流
    │   └─ 写多 → 加版本号防覆盖
    │
    └─ 否（写少读多且强一致）
        └─ 不用缓存，或缓存做物化视图
```

---

## 九、要做的监控指标

| 指标                       | 说明                                       |
| -------------------------- | ------------------------------------------ |
| 缓存命中率                 | < 95% 通常说明 TTL 或回源策略有问题        |
| 缓存与 DB 双读对比抽样     | 定时抽样比对，发现不一致告警               |
| binlog 订阅延迟            | 兜底链路是否健康                           |
| 删除失败次数               | Redis 删除/更新失败的计数                  |
| 热点 key 监控              | 防止单 key QPS 把 Redis 单分片打挂         |

---

## 十、一句话总结

> **正常业务用 Cache Aside + 短 TTL 即可；高并发系统用 Cache Aside + binlog 订阅兜底；强一致场景就不要用缓存。**

工程上没有"完美的一致性"，只有"足够小的不一致窗口 + 足够好的监控告警 + 可接受的运维成本"三者的平衡。

