#  Redis 缓存淘汰策略
{docsify-updated}

> https://redis.io/docs/reference/eviction/

- [Redis 缓存淘汰策略](#redis-缓存淘汰策略)
  - [设置最大内存使用量](#设置最大内存使用量)
  - [配置缓存淘汰策略](#配置缓存淘汰策略)

### 设置最大内存使用量
`maxmemory` 配置指令可将 Redis 配置为使用指定内存量的数据集。你可以使用 redis.conf 文件，或在运行时使用 CONFIG SET 命令来设置配置指令。
```
maxmemory 100mb
```
将 maxmemory 设置为零，则没有内存限制。这是 64 位系统的默认行为，而 32 位系统使用 3GB 的隐式内存限制。  
当达到指定的内存量时，eviction policies 配置的策略决定了默认行为。Redis 可以为可能导致使用更多内存的命令返回错误，也可以在每次添加新数据时驱逐一些旧数据，以返回指定限制。

### 配置缓存淘汰策略
当达到 `maxmemory` 限制时，Redis 所遵循的具体行为是通过 `maxmemory-policy` 配置指令来配置的。

有以下策略可供选择：
+ `noeviction`：达到内存限制时不保存新值。当数据库使用复制时，这适用于主数据库
+ `allkeys-lru`：保留最近使用的键值；删除最近最少使用（LRU）的键值
+ `allkeys-lfu`：保留经常使用的密钥；删除最不经常使用（LFU）的密钥
+ `volatile-lru`：删除过期字段设置为 true 的最近最少使用的密钥。
+ `volatile-lfu`：删除过期字段设置为 true 的最不常用密钥。
+ `allkeys-random`：随机删除密钥，为新添加的数据腾出空间。
+ `volatile-random`：随机删除过期字段设置为 true 的密钥。
+ `volatile-ttl`：删除过期字段设置为 true 且剩余生存时间（TTL）值最短的密钥。

如果没有符合前提条件的key需要驱逐，则 `volatile-lru、volatile-lfu、volatile-random 和 volatile-ttl` 策略的行为与 noeviction 保持一致。

根据应用程序的访问模式选择正确的驱逐策略非常重要，不过，你可以在运行应用程序时重新配置策略，并使用 Redis INFO 输出监控缓存未命中和命中的次数，以调整设置。

一般来说，这是一条经验法则：
+ 当你预期请求的流行度呈幂律分布时，请使用 `allkeys-lru` 策略。也就是说，你预计某个子集元素的访问频率会远远高于其他元素。如果你不确定，这是一个不错的选择。
+ 如果是循环访问，所有密钥都会被连续扫描，或者预期分布是均匀的，则使用 `allkeys-random`。
+ 如果你希望在创建缓存对象时，通过使用不同的 TTL 值为 Redis 提供过期提示，那么请使用 `volatile-ttl`。

volatile-lru 和 volatile-random 策略主要适用于既想使用单个实例进行缓存，又想拥有一组持久键的情况。不过，要解决这种问题，通常最好运行两个 Redis 实例。

值得注意的是，为密钥设置过期值会占用内存，因此使用 allkeys-lru 这样的策略会更节省内存，因为在内存压力下，不需要过期配置来驱逐密钥。