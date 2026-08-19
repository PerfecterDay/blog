# Redis高可用
{docsify-updated}

<center><img src="/pic/redis_ha.jpg" alt=""></center>

## Redis 复制

在分布式系统中为了解决单点问题，通常会把数据复制多个副本部署到其他机器，满足故障恢复和负载均衡等需求。Redis也是如此，它为我们提供了复制功能，实现了相同数据的多个Redis副本。复制功能是高可用Redis的基础，后面章节的哨兵和集群都是在复制的基础上实现高可用的。

参与复制的Redis实例划分为主节点（master）和从节点（slave）。默认情况下，Redis都是主节点。每个从节点只能有一个主节点，而主节点可以同时具有多个从节点。复制的数据流是单向的，只能由主节点复制到从节点。配置复制的方式有以下三种：

1. 在配置文件中加入 `slaveof {masterHost} {masterPort}` 随Redis启动生效。
2. 在 redis-server 启动命令后加入 `--slaveof {masterHost} {masterPort}` 选项，启动后生效。
3. 直接使用命令：`slaveof {masterHost} {masterPort}` ，命令执行后生效。

主从节点复制成功建立后，可以使用 `info replication` 命令查看复制相关状态。

slaveof命令不但可以建立复制，还可以在从节点执行 `slaveof no one` 来断开与主节点复制关系。从节点断开复制后并不会抛弃原有数据，只是无法再获取主节点上的数据变化。把当前从节点对主节点的复制切换到另一个主节点，执行 `slaveof {newMasterIp} {newMasterPort}` 命令即可，切主后从节点会清空之前所有的数据，线上人工操作时小心 slaveof 在错误的节点上执行或者指向错误的主节点。



## Redis 哨兵
```
    S1 ─────┐
    S2 ─────┼────> Master A
    S3 ─────┘         │
                        │
                ┌────┴────┐
                ▼         ▼
                Replica B  Replica C
```

1. `SENTINEL get-master-addr-by-name mymaster`
2. `info replication`
      ```
         10.5.160.73:6380> info replication
         # Replication
         role:master
         connected_slaves:2
         min_slaves_good_slaves:2
         slave0:ip=10.5.160.37,port=6380,state=online,offset=1338195327,lag=1
         slave1:ip=10.5.160.36,port=6380,state=online,offset=1338195327,lag=1
         master_failover_state:no-failover
         master_replid:79abe0e45168baf9b9ff8acd3d17917f4e3bca09
         master_replid2:0000000000000000000000000000000000000000
         master_repl_offset:1338195327
         second_repl_offset:-1
         repl_backlog_active:1
         repl_backlog_size:268435456
         repl_backlog_first_byte_offset:1069759872
         repl_backlog_histlen:268435456
      ```
