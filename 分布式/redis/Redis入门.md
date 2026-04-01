# Redis 入门
{docsify-updated}

Redis是一种基于键值对（key-value）的NoSQL数据库，与很多键值对数据库不同的是，Redis中的值可以是由string（字符串）、hash（哈希）、list（列表）、set（集合）、zset（有序集合）、Bitmaps（位图）、HyperLogLog、GEO（地理信息定位）等多种数据结构和算法组成，因此Redis可以满足很多的应用场景，而且因为Redis会将所有数据都存放在内存中，所以它的读写性能非常惊人。不仅如此，Redis还可以将内存的数据利用快照和日志的形式保存到硬盘上，这样在发生类似断电或者机器故障的时候，内存中的数据不会“丢失”。除了上述功能以外，Redis还提供了键过期、发布订阅、事务、流水线、Lua脚本等附加功能。总之，如果在合适的场景使用好Redis，它就会像一把瑞士军刀一样所向披靡。

## Redis 安装
+ CentOS 安装实战
	1. 官网下载 redis 源码包： https://redis.io/download/；
	2. 拷贝到linux 服务器后，执行 make install，安装完成
	3. 拷贝源码包中的 redis.conf 到 dev/redis 目录 

## Redis Server 启动与连接
<center>
<img src="pics/redis-exe.png" width="60%" alt="">
</center>

1. 默认配置启动：`# redis-server`
2. 指定配置文件启动: `# redis-server /opt/redis/redis.conf`
3. 运行时通过命令行参数指定配置启动：`# redis-server --configKey1 configValue1 --configKey2 configValue2`

Redis目录下都会有一个redis.conf配置文件，里面就是Redis的默认配置，通常来讲我们会在一台机器上启动多个Redis，并且将配置集中管理在指定目录下，而且配置不是完全手写的，而是将redis.conf作为模板进行修改。一些基础配置如下：
+ `bind` : 绑定IP地址,默认情况只绑定了 127.0.0.1， 也就是说只能从本机访问
+ `port` : 监听端口（Redis的默认端口是6379）
+ `logfile` : 日志文件存放位置
+ `dir` : Redis 工作目录（存放持久化文件和日志文件）
+ `daemonize`: 是否以守护进程的方式启动
+ `protected-mode yes` : 保护模式开关
+ `dbfilename`: 备份文件的文件名字
+ `requirepass foobared`:设置密码


## redis-cli
+ `redis-cli -h {host} -p {port} -a {password}` ：使用主机、端口和密码以交互式连接 redis 服务
+ `redis-cli -h {host} -p {port} -a {password} {command}` ：使用主机、端口和密码以命令式连接 redis 服务并执行一条命令
+ `redis-cli -n {dbnum} {command}` ：在 <dbnum> 指定的数据库中执行操作
+ `redis-cli -u redis://{user}:{password}@{host}:{port}/{dbnum}` ：以URL的方式指定参数， `redis-cli -u redis://LJenkins:p%40ssw0rd@redis-16379.hosted.com:16379/0 PING`
+ `redis-cli -r {count} -i {delay seconds} {command}` : 持续重复执行某个操作 {count} 次，每两次间隔 {delay} 秒，{count}为 -1时代表一直执行
+ `redis-cli --stat -i <interval>` : 持续查看服务器状态， `-i <interval>` 指定间隔时间
+ `redis-cli CLIENT LIST` : 查看当前的连接情况及使用的DB信息

使用上下箭头可以查看 redis-cli 中执行的历史命令，历史命令保存在 `$HOME/.rediscli_history` 文件中。如果想保存在别的文件中，使用环境变量 `REDISCLI_HISTFILE` 定义指定位置即可，如果不想保留历史记录，将 `REDISCLI_HISTFILE` 设置为 `/dev/null`。

### redis-cli 的配置
有两种方法可以自定义 redis-cli 的行为。CLI 会在启动时加载主目录中的 `.redisclirc` 文件。可以使用 `REDISCLI_RCFILE` 环境变量设置为其他路径，从而覆盖配置文件的默认位置。也可以在 CLI 会话期间设置首选项，在这种情况下，首选项仅在会话期间有效。

使用特殊命令 `:set` 来配置 CLI。这些指令既可以保存在 `.redisclirc` 文件中，也可以直接在 CLI 交互模式下输入即可生效。例子：
+ `:set hints` - enables syntax hints
+ `:set nohints` - disables syntax hints

### 交互模式
+ `auth {user} {password}` : 使用账号密码认证
+ `select {dbnum}` : 选定使用的数据库
+ `connect {host} {port}` : 连接指定的数据库
+ `{num} {comman}` : 重复执行命令若干次， `5 INCR mycounter`

### 查看帮助
`HELP @<category>` 显示有关给定类别的所有命令。 <category>包括: 
+ @generic
+ @string
+ @list
+ @set
+ @sorted_set
+ @hash
+ @pubsub
+ @transactions
+ @connection
+ @server
+ @scripting
+ @hyperloglog
+ @cluster
+ @geo
+ @stream

`HELP <command>` 查看指定命令的帮助


### 关闭 Redis 服务
1. `redis-cli shutdown`：断开与客户端的连接、持久化文件生成，是一种相对优雅的关闭方式。
2. shutdown还有一个参数，代表是否在关闭Redis前，生成持久化文件：`redis-cli shutdown nosave|save`
3. 除了可以通过shutdown命令关闭Redis服务以外，还可以通过kill进程号的方式关闭掉Redis，但是不要粗暴地使用kill-9强制杀死Redis服务，不但不会做持久化操作，还会造成缓冲区等资源不能被优雅关闭，极端情况会造成AOF和复制丢失数据的情况。

## 常用命令

1. 配置命令
   1. 查看所有配置： `config get *`
   2. 查看指定配置：`config get XXX`
   3. 设置config： `config set xxx xxx`
2. 键通用命令
   1. 查看所有的键： `keys *`，会遍历所有的键，谨慎使用。
   2. 查看部分匹配键： `keys info:*`,查看 info: 开头前缀的键
   3. 删除指定键：`redis-cli -h IP -p PORT -a PASSWORD keys 'key*' | xargs redis-cli -h IP  -p PORT -a PASSWORD del`
   4. 清空所有键值对: `FLUSHALL` 
   5. 键总数： `dbsize` ,不会遍历所有键，而是直接获取 Redis内置的键总数变量的值
   6. 检查键是否存在： `exists key`， 存在返回1，否则返回0
   7. 删除键： `del key1 key2 ...`, 返回成功删除键的个数
   8. 键过期： `expire key seconds` ,超过过期时间后，键会自动删除
   9. 键剩余过期时间： `ttl key`，返回大于等于0的整数，代表剩余过期时间；如果没有设置过期时间，返回-1；键不存在，返回-2
   10. 查看键对应的值的数据类型： `type key`，键不存在返回 none
   11. 键重命名： `rename key newkey`, 如果 newkey 已经存在，那么他的值将会被 key 的值覆盖
   12. `renamenx key newkey`： 只有 newkey 不存在时才会重命名成功，由于重命名键期间会执行del命令删除旧的键，如果键对应的值比较大，会存在阻塞Redis的可能性，这点不要忽视。
   13. 随机返回一个键： `randomkey`

3. 信息查看命令
   1. 查看连接的客户端：`redis 127.0.0.1:6379> CLIENT LIST` 


## Jedis 简介
1. 直接构造操作 Jedis 
   ```
   # 1. 生成一个Jedis对象，这个对象负责和指定Redis实例进行通信
   Jedis jedis = new Jedis("127.0.0.1", 6379);
   # 2. jedis执行set操作
   jedis.set("hello", "world");
   # 3. jedis执行get操作, value="world"
   String value = jedis.get("hello");
   ```
   还有一个包含4个参数的构造函数是比较常用的：
   `Jedis(final String host, final int port, final int connectionTimeout, final int
   soTimeout)`
   + host：Redis实例的所在机器的IP。
   + port：Redis实例的端口。
   + connectionTimeout：客户端连接超时。
   + soTimeout：客户端读写超时。

   Jedis 对 Redis 五种数据结构的操作示例：
   ```
   // 1.string
   // 输出结果：OK
   jedis.set("hello", "world");
   // 输出结果：world
   jedis.get("hello");
   // 输出结果：1
   jedis.incr("counter");
   // 2.hash
   jedis.hset("myhash", "f1", "v1");
   jedis.hset("myhash", "f2", "v2");
   // 输出结果：{f1=v1, f2=v2}
   jedis.hgetAll("myhash");
   // 3.list
   jedis.rpush("mylist", "1");
   jedis.rpush("mylist", "2");
   jedis.rpush("mylist", "3");
   // 输出结果：[1, 2, 3]
   jedis.lrange("mylist", 0, -1);
   // 4.set
   jedis.sadd("myset", "a");
   jedis.sadd("myset", "b");
   jedis.sadd("myset", "a");
   // 输出结果：[b, a]
   jedis.smembers("myset");
   // 5.zset
   jedis.zadd("myzset", 99, "tom");
   jedis.zadd("myzset", 66, "peter");
   jedis.zadd("myzset", 33, "james");
   // 输出结果：[[["james"],33.0], [["peter"],66.0], [["tom"],99.0]]
   jedis.zrangeWithScores("myzset", 0, -1);
   ```
2. Jedis 连接池
   Jedis提供了JedisPool这个类作为对Jedis的连接池，同时使用了Apache的通用对象池工具common-pool作为资源的管理工具，下面是使用JedisPool操作Redis的代码示例：
   ```
   // common-pool连接池配置，这里使用默认配置，后面小节会介绍具体配置说明
   GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
   // 初始化Jedis连接池
   JedisPool jedisPool = new JedisPool(poolConfig, "127.0.0.1", 6379);

   //从连接池获取 Jedis 连接对象
   Jedis jedis = null;
   try {
      // 1. 从连接池获取jedis对象
      jedis = jedisPool.getResource();
      // 2. 执行操作
      jedis.get("hello");
   } catch (Exception e) {
      logger.error(e.getMessage(),e);
   } finally {
      if (jedis != null) {
         // 如果使用JedisPool，close操作不是关闭连接，代表归还连接池
         jedis.close();
      }
   }
   ```


## 设置过期时间
1. `expire key 10000`: 设置 key 过期时间 1000s
2. `ttl key`: 查看 key 剩余存活时间，单位s，-1 表示没有设置过期时间，-2表示key不存在
3. `persist key`: 取消key 的过期时间设置
   
下边是 Jedis 连接池的配置参数：
   <center> <img src="pics/jedispool-config.png" width="60%"></center>
  

