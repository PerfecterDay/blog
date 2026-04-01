# Redis 持久化
{docsify-updated}

- [Redis 持久化](#redis-持久化)
    - [RDB](#rdb)
        - [RDB 触发方式](#rdb-触发方式)
        - [RDB快照保存过程](#rdb快照保存过程)
        - [RDB文件的处理](#rdb文件的处理)
        - [RDB的优缺点](#rdb的优缺点)
    - [AOF](#aof)
        - [AOF  刷盘配置](#aof--刷盘配置)
        - [AOF重写](#aof重写)


Redis 支持两种持久化方式：
+  RDB(Redis Database File),Snapshoting (快照，默认方式)
+  Append-only file (AOF)

### RDB
RDB是默认的持久化方式。这种方式是就是将内存中数据以快照的方式写入到二进制文件中,默认的文件名为 `dump.rdb` 。

##### RDB 触发方式

触发RDB持久化过程分为手动触发和自动触发。

1. 手动触发
   手动触发分别对应 save 和 bgsave 命令：
   + save 命令：对于单线程的 Redis 服务器，会阻塞服务器，直到 RDB 过程完成为止，对于内存中存放大量数据的实例会造成长时间阻塞，线上环境不建议使用。
   + bgsave 命令：Redis 进程执行 fork 操作创建子进程，RDB过程由子进程负责，完成后自动结束，阻塞只发生在 fork 阶段，一般时间很短。
2. 自动触发
   下述场景会自动触发 RDB 持久化机制：
   1. 通过配置设置了自动做快照持久化的方式。我们可以配置 redis 在 n 秒内如果超过 m 个 key 被修改就自动做快照（bgsave），下面是默认的快照保存配置：
      + `save 900 1` #900 秒内如果超过 1 个 key 被修改，则发起快照保存
      + `save 300 10` #300 秒内容如超过 10 个 key 被修改，则发起快照保存
      + `save 60 10000` #60 秒内容如超过 10000 个 key 被修改，则发起快照保存
   2. 如果从节点执行全量复制操作，主节点会自动执行 bgsave 生成 RDB 文件发送给从节点
   3. 执行 debug reload 命令重新加载 Redis 时，也会自动触发
   4. 默认情况下执行 shutdown 命令时，如果没有开启 AOF 持久化功能则自动执行 bgsave 。

##### RDB快照保存过程

1. redis 调用 fork后,于是有了子进程和父进程。
2. 父进程继续处理 client 请求，子进程负责将内存内容写入到临时文件。由于 os 的实时复制机制（ copy on write)父子进程会共享相同的物理页面，当父进程处理写请求时 os 会为父进程要修改的页面创建副本，而不是写共享的页面。所以子进程地址空间内的数据是 fork 时刻整个数据库的一个快照。
3. 当子进程将快照写入临时文件完毕后，用临时文件替换原来的快照文件，然后子进程退出。 client 也可以使用 `save` 或者 `bgsave` 命令通知 redis 做一次快照持久化。 save 操作是在主线程中保存快照的，由于 redis 是用一个主线程来处理所有 client 的请求，这种方式会阻塞所有client 请求。所以不推荐使用。另一点需要注意的是，每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步变更数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘 io 操作，可能会严重影响性能。

##### RDB文件的处理

RDB文件保存在`dir`配置指定的目录下，文件名通过 `dbfilename` 配置指定。可以通过执行 `config set dir{newDir}` 和 `config set dbfilename {newFileName}` 运行期动态执行，当下次运行时RDB文件会保存到新目录。

Redis默认采用LZF算法对生成的RDB文件做压缩处理，压缩后的文件远远小于内存大小，默认开启，可以通过参数 `config set rdbcompression {yes|no}` 动态修改。

如果Redis加载损坏的RDB文件时拒绝启动，可以使用Redis提供的 `redis-check-dump` 工具检测RDB文件并获取对应的错误报告。

##### RDB的优缺点

1. RDB的优点：
   + RDB是一个紧凑压缩的二进制文件，代表Redis在某个时间点上的数据快照。非常适用于备份，全量复制等场景。比如每6小时执行bgsave备份，并把RDB文件拷贝到远程机器或者文件系统中（如hdfs），用于灾难恢复。
   + Redis加载RDB恢复数据远远快于AOF的方式。
2. RDB的缺点：
   + RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高。
   + RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，存在老版本Redis服务无法兼容新版RDB格式的问题。


### AOF
首先通过 `appendonly yes` 启用 aof 持久化方式。AOF 以独立日志的方式记录每次写命令。重启时再重新执行AOF文件中的命令达到恢复数据的目的。AOF的主要作用是解决了数据持久化的实时性，目前已经是Redis持久化的主流方式。
由于快照方式是在一定间隔时间做一次的，所以如果 redis 意外 down 掉的话，就会丢失最后一次快照后的所有修改。如果应用要求不能丢失任何修改的话，可以采用 aof 持久化方式。

##### AOF  刷盘配置

aof 比快照方式有更好的持久化性能，是由于在使用 aof 持久化方式时,redis 会将每一个收到的写命令都通过 write 函数追加到日志文件中(默认是 appendonly.aof)。当 redis 重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当然由于 os 会在内核中缓存（AOF缓冲区） write 做的修改，所以可能不是立即写到磁盘上。这样 aof 方式的持久化也还是有可能会丢失部分修改。不过我们可以通过配置文件告诉 redis 我们想要通过 fsync 函数强制 os 写入到磁盘的时机。有三种方式如下（默认是：每秒 fsync 一次）：

1. `appendfsync always` //收到写命令写入aof_buf后调用 fsync 就立即写入磁盘，fsync 完成后线程返回。最慢，但是保证完全的持久化
2. `appendfsync everysec` //默认方式，，命令写入 aof_buf 后调用 write 操作，write 操作完成线程返回。fsync 同步文件由专门线程每秒调用一次，在性能和持久化方面做了很好的折中
3. `appendfsync no` //命令写入 aof_buf 后调用系统 write 操作，不对 AOF 文件做 fsync 同步，完全依赖 os，性能最好,持久化没保证

##### AOF重写

aof 的方式也同时带来了另一个问题。持久化文件会变的越来越大。AOF重写能压缩文件体积，有以下原因：

+ 多条命令合并。例如我们调用 incr test 命令 100 次，文件中必须保存全部的 100 条命令，其实有 99 条都是多余的。因为要恢复数据库的状态其实文件中保存一条 set test 100 就够了。
+ 进程内已经超时的数据不会再写入文件
+ 旧的AOF文件包含无效命令，如 `del key1,hdel key2,srem keys,set a 111, set a 222`等。重写使用进程内数据直接生成，这样新的AOF文件只保留最终数据的写入命令。

AOF重写降低了文件占用空间，而且也可以更快地被 Redis 加载。为了压缩 aof 的持久化文件， redis 提供了两种方式：
1. 手动触发：`bgrewriteaof` 命令。收到此命令 redis 将使用与快照类似的方式将内存中的数据以命令的方式保存到临时文件中，最后替换原来的文件。
2. 自动触发：根据`auto-aof-rewrite-min-size`和`auto-aof-rewrite-percentage`参数确定自动触发时机。
   + `auto-aof-rewrite-min-size`：表示运行AOF重写时文件最小体积，默认为64MB。
   + `auto-aof-rewrite-percentage`：代表当前AOF文件空间（aof_current_size）和上一次重写后AOF文件空间（aof_base_size）的比值。
   自动触发时机=`aof_current_size > auto-aof-rewrite-minsize &&（aof_current_size-aof_base_size）/aof_base_size >= auto-aof-rewritepercentage` 
   其中aof_current_size和aof_base_size可以在info Persistence统计信息中查看。
