#  mysql innodb 存储引擎
{docsify-updated}

## 存储引擎
MySQL 服务器把数据的存储和提取操作都封装到了一个叫 **存储引擎** 的模块里。我们知道表是由一行一行的记录组成的，但这只是一个逻辑上的概念，物理上如何表示记录，怎么从表中读取数据，怎么把数据写入具体的物理存储器上，这都是存储引擎负责的事情。为了实现不同的功能， MySQL 提供了各式各样的存储引擎 ，不同存储引擎管理的表具体的存储结构可能不同，采用的存取算法也可能不同。

为了管理方便，人们把 **连接管理** 、 **查询缓存** 、 **语法解析** 、 **查询优化** 这些并不涉及真实数据存储的功能划分为 MySQL server 的功能，把真实存取数据的功能划分为 **存储引擎** 的功能。各种不同的存储引擎向上边的 MySQL server 层提供统一的调用接口（也就是存储引擎API），包含了几十个底层函数，像"读取索引第一条内容"、"读取索引下一条内容"、"插入记录"等等。所以在 MySQL server 完成了查询优化后，只需按照生成的执行计划调用底层存储引擎提供的API，获取到数据后返回给客户端就好了。

使用 `SHOW ENGINES;` 可以查看服务器支持的存储引擎信息。

## Innodb 体系架构

<center><img src="pics/inno_engine.jpg" width="60%"></center>

#### 后台线程
1. Master Thread  
Master Thread 是核心线程，主要负责将缓冲池中的缓存异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、undo 页的回收等。

2. IO Thread  
Innodb 存储引擎大量使用异步 AIO 来处理 IO请求，这样做可以提高数据库的性能。在 1.0 版本之前，有4个IO 线程，分别是 read、write、insert buffer 和 log IO thread 。从1.0.x 版本开始 read 和 write 的线程数增大到4个，并且可以通过 `innodb_read_io_threads` 和 `innodb_write_io_threads` 参数进行设置。可以通过 `show engine innodb status` 来查看 innodb 引擎的运行状态。

3. Purge Thread  
Purge Thread 用来回收已经提交的事务的 undo log页。 可以在配置文件中配置 `innodb_purge_threads=4` 来配置回收线程的数量。

4. Page Cleaner Thread  
该线程的主要作用是将脏页的刷新操作放入到本线程中来，减轻 Master thread 的工作，提高系统性能。

#### 内存
1. 缓冲池  
    为了弥补CPU和磁盘之间的速度差距，基于磁盘的数据库系统普遍使用缓冲池技术来提高数据库整体性能。

    缓冲池就是一块内存区域，数据库读取页时，首先判断磁盘页是否在内存池中，如果在，就直接读取内存中的页，否则从磁盘读取该页并将页加载到缓冲池中。  
    对于页的修改操作，也是首先修改缓存池中的页，然后以一定的频率刷新回磁盘。注意刷新到磁盘的操做不是每次修改页时都会发生的，而是通过 Checkpoint 的机制来刷新的。
    <center><img src="/pics/innodb-memory-pool.jpg" width="50%"></center>

    缓冲池的大小设置参数： `innoodb_buffer_pool_size` 。从 Innodb 1.0.x 开始，允许有多个缓冲池实例，通过 `innodb_buffer_pool_instances` 参数设置缓冲池个数。这样做的好处是减少数据库内部的资源竞争，提高数据库的并发能力。

2. LRU List、Free List和 Flush List  
    数据库中的缓冲池是通过 LRU （Latest Recent Used,最近最少使用）算法来管理的。最频繁使用的页存储在LRU列表的前段，最少使用的存放在尾端。缓冲池内存不够时，首先释放尾端的页。Innodb对传统LRU算法作了一些优化，使用了 midpoint insertion strategy 。新读取到的页不是存放在 LRU 列表的首部，而是放入到列表的 midpoint 位置。 midpoint 位置由参数 `innodb_old_blocks_pct` 设置，设置的是从列表尾端开始的一个百分比。例如， `innodb_old_blocks_pct=37` 代表新读取的页插入到列表尾端的37%的位置。 midpoint 之后的列表称为 old 列表，之前的称为 new 列表。`innodb_old_blocks_time` 是管理 LRU 列表的另一个参数，用来表示页被读取到 midpoint 位置后需要等待多久才能被加载到 new 列表。

    LRU 列表管理的是已经读取的页，其实就是已经分配的内存。缓冲池中的空闲内存由 Free 列表管理。当数据库刚刚启动时，LRU 列表是空的，即还没任何页被读取到缓冲池。当需要从缓冲池中分配页时，首先从 Free 列表中查看是否有可用的空闲页，如果有就将该页从Free 列表删除，添加到 LRU 列表中，否则根据 LRU 算法淘汰 LRU 列表末尾的页。一般来说，LRU 列表大小+Free 列表大小 &lt; 缓存池大小，因为缓冲池中的页还会被分配给自适应索引、Lock信息、Insert Buffer等页，这部分页不在LRU列表中维护。

    LRU 列表中的页被修改后，称为脏页，FLUSH 列表用来管理脏页。脏页既存在于 LRU 列表也存在于 FLUSH 列表。LRU 列表用来管理缓冲池中页的可用性，FLUSH 列表用来管理页的刷新操作。

    页从 old 列表移到 new 列表的操作称为：page made young 。而因为 `innodb_old_blocks_time` 的设置而没有完成移动的操作称为：page not made young。使用 `show engine innodb status` 查看 innodb 状态时一些参数代表的含义如下：

    + Buffer pool size : 指示缓冲池一共有多少页，（默认一页为 16KB）
    + Free buffers： Free 列表中页的数量
    + Database pages： LRU 列表中页的数量
    + Buffer pool hit rate: 缓冲池的命中率（缓存命中率），通常该值不小于95%。
    + LRU len: LRU 列表中页的数量包含 unzip_LRU 列表中页数量，unzip_LRU len: 压缩页（非16KB页）列表中页的数量，
    + Modified db pages：脏页数量

3. 重做日志缓冲区
    Innodb重做日志缓冲用来缓存重做日志信息，然后按照一定频率将其刷新到重做日志文件。重做日志缓冲区的大小由 `innodb_log_buffer_size` 设置。重做日志缓冲区一般不需要设置很大，因为以下三种情况会将日志刷新到重做日志文件中：
    1. Master Thread 每秒刷新一次重做日志到重做日志文件
    2. 每个事务提交时
    3. 当重做日志缓冲区剩余空间小于一半时

### Checkpoint技术
缓冲池的设计目的是为了缓解CPU速度与磁盘速度的鸿沟，因此页的操作首先都是在缓冲池中完成的。如果一条DML语句如 update 或 delete 改变了页中的记录，那么此时缓冲池中页的版本比磁盘中的新，此时该页称为脏页。数据库需要将脏页写回磁盘（刷盘）。

如果将脏页刷新到磁盘之前，数据库发生了宕机，那么数据将会丢失。为了防止这种情况，当前的事务数据库普遍使用了 Write Ahead Log 策略，即当事务提交时，在修改数据页之前，先在重做日志中记录相关信息。这样如果修改数据失败时，利用重做日志就可以完成数据库恢复。注意到，如果事务写完重做日志，然后修改内存缓冲区中对应页都成功，且脏页刷盘也成功了，事务执行就算是成功了，那么这部分重做日志实际上就已经没用了，此时可以记一个 checkpoint，checkpoint 之前的日志是可以被覆盖。

Checkpoint 技术主要用来解决以下几个问题：
1. 缩短数据库恢复时间  
假如没有 checkpoint ，那么就要应用所有重做日志，这可能需要大量时间。有了 checkpoint 的话，只要用某个 checkpoint 之后的重做日志来恢复即可。
2. 缓冲池不够用时，将脏页刷新到磁盘  
当缓冲池不够用时，我们可以将脏页刷新到磁盘并且释放这些脏页所占的内存，checkpoint 可以标识脏页。
3. 重做日志不可用时，刷新脏页  
重做日志不可用是指，日志已经到达指定大小，但是重做日志中的内容仍然不能被覆盖，此时，必须强制将脏页刷新到磁盘，并产生新的 checkpoint ，释放重做日志。

对于 Innodb 存储引擎来说，其是通过 LSN（Log Sequence Number)来标记版本的。LSN 是一个8字节大小的数字，记录的单位是字节。每个页有LSN，重做日志中也有LSN， Checkpoint 也有 LSN 。Checkpoint 所做的事情就是将缓冲池中的脏页刷新到磁盘。不同之处在于每次刷新多少页到磁盘，每次从哪里取脏页，以及什么时间触发 Checkpoint 。在 Innodb 内部有两种 Checkpoint，分别为：
1. Sharp Checkpoint  
默认情况下（参数 `innodb_fast_shutdown=1`），发生在数据库关闭时，将所有的脏页刷新到磁盘。
2. Fuzzy Checkpoint  
这种情况下只会刷新部分脏页到磁盘。通常在如下几种情况下发生：
    + Master Thread Checkpoint  
        Master Thread 会以每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的脏页回磁盘。
    + FLUSH_LRU_LIST Checkpoint  
        当从LRU List 末尾移除页时，如果是脏页，需要进行Checkpoint。
    + Async/Sync Flush Checkpoint
        当重做日志不可用时，需要从脏页列表中将一些脏页刷新回磁盘。这操作自 1.2.x之后放入到 Page Cleaner Thread 中完成。
    + Dirty Page too much Checkpoint
        当缓冲池中脏页数量达到一定比例时（`innodb_max_dirty_page_pct` 参数设置），innodb 存储引擎会强制 Checkpoint，释放缓冲池内存. 

### Innodb 关键特性
1. 插入缓冲
2. 两次写
    考虑这样一种情况，当对一个 16KB 的页进行写入时，写入 4KB 时，发生了宕机，这种情况称为部分写失效。这时能通过重做日志恢复吗？重做日志记录的是对页的物理操作，如偏移量800写'aaa'记录，如果这个页（这个页是内存中的页，重做日志记录的也是基于内存页来说的而不是磁盘页，因为内存页是最新的而磁盘页不是，如在刷盘前，连续对一个页进行两次或多次更新操作，重做日志只能是基于内存页）本身已经损坏，再对其进行重做是没有意义的。也就是说，应用重做日志之前，用户需要一个副本，当写入失效发生，先用副本还原该页，然后再应用重做日志重做。

    因此，innodb在刷盘时，并不是直接将脏页写入到磁盘页中，而是先将脏页写到一个2MB大小的双写缓冲区，然后将其写入到共享表空间中的2M双写区，这是顺序写入，开销不是很大。写到双写区后，在将双写缓冲区中的页写入到磁盘页中，这是离散写的。
    ![双写](../pics/innodb-double-write.png)
    如果，将脏页刷新到磁盘的过程中出现错误，可以先用双写区中的副本还原磁盘上的页，然后用重做日志对其进行重做。
    使用`show global status like '%innodb_dbwr%'\G` 可以观察数据库双写的运行情况。

3. 自适应哈希索引
4. 异步 IO
5. 刷新邻接页