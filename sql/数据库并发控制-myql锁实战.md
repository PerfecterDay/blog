# mysql 锁实战
{docsify-updated}

## data_locks 表
`performance_schema.data_locks` 是 MySQL 8.0+ 中的一个非常有用的诊断系统表，用于查看当前数据库中哪些事务持有哪些锁、等待哪些锁，从而帮助你分析：
+ 死锁（deadlock）
+ 锁等待（lock wait）
+ 并发冲突（如阻塞、事务等待）
+ 行级锁状态（InnoDB）

通常与 `performance_schema.data_lock_waits` 和 `performance_schema.threads` 联用，用于分析阻塞链.


1. 查看	mysql 锁信息
```
select * from performance_schema.data_locks;
```

`data_locks` 表字段详解：
| 字段名                  | 含义说明                                                                                         |
|--------------------------|--------------------------------------------------------------------------------------------------|
| ENGINE                   | 锁所属的存储引擎，通常为 InnoDB。                                                               |
| ENGINE_LOCK_ID           | 存储引擎内部生成的锁 ID，与 ENGINE 组合后唯一，用于锁的关联。ID 格式内部定义，不要依赖其结构。   |
| ENGINE_TRANSACTION_ID    | 请求锁的事务 ID，若事务仅读，则可能不是正式 ID；用于关联 `INFORMATION_SCHEMA.INNODB_TRX.TRX_ID` 查看该事务详情。    |
| THREAD_ID                | 发起此锁请求的线程 ID，可连接 `performance_schema.threads` 获取线程/会话信息。                    |
| EVENT_ID                 | 创建此锁数据结构的事件 ID。可用于关联 `performance_schema.events_*` 等表（如 `events_statements_...`）。              |
| OBJECT_SCHEMA            | 锁所作用的数据库名称。                                                                           |
| OBJECT_NAME              | 被锁定的表名称。                                                                                 |
| PARTITION_NAME           | 被锁定的分区名（若表分区），无分区则为 `NULL` 。                                                  |
| SUBPARTITION_NAME        | 被锁定的子分区名（若存在），否则为 `NULL` 。                                                      |
| INDEX_NAME               | 所用索引的名称。InnoDB 至少会有 `GEN_CLUST_INDEX` （聚簇索引）。                                   |
| OBJECT_INSTANCE_BEGIN    | 内存中此锁对象的地址，可用于唯一识别锁实例。                                                    |
| LOCK_TYPE                | 锁类型，如 `RECORD`（行锁）或 `TABLE`（表锁）。                                                     |
| LOCK_MODE                | 锁模式，如 `S, X, IS, IX, AUTO_INC` 等。`GAP` 后缀表示间隙锁。                                   |
| LOCK_STATUS              | 锁状态：`GRANTED` 表示已获取，`WAITING` 表示正在等待获取。                                          |
| LOCK_DATA                | 仅在 `RECORD` 锁时显示所锁定的数据内容（如主键或聚簇键值），其他锁该字段为 `NULL`。                 |

### LOCK_TYPE
1. `RECORD` : 行锁（Row Lock）：表示锁定了某一行或一组记录。大多数事务锁都是这个类型。基于索引进行的行级加锁。
2. `TABLE` : 表锁：表示锁定了整个表，例如在 DDL 操作、意向锁、metadata 锁 等场景中出现。InnoDB 支持的表级锁类型较少。
3. `AUTO_INC` : 自增锁（Auto Increment Lock）：用于保护带有 AUTO_INCREMENT 的列，防止多个事务同时生成相同的 ID。是一种内部表级特殊锁。
4. `INTENTION` : 意向锁：如 IS（意向共享锁）、IX（意向排他锁）都属于这一类。标明某事务想要获取表中某行的某种锁，但并不直接阻塞其他操作。
5. `METADATA` : 元数据锁（Metadata Lock）：锁定表的结构信息，常出现在 DDL 与查询/事务并发时。严格来说这是 metadata locking 层的锁，有时能通过 `INFORMATION_SCHEMA.METADATA_LOCKS` 观察。并不一定出现在 `data_locks` 中。

### LOCK_MODE
| 锁模式 (LOCK_MODE)         | 含义说明                                                                                       |
|----------------------------|------------------------------------------------------------------------------------------------|
| S (Shared)                 | **共享锁**。允许多个事务同时读同一数据，但不能写。常见于：`SELECT ... FOR SHARE`。     |
| X (Exclusive)              | **排他锁**。仅允许当前事务访问该数据，其他事务不能读写。常见于：`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE`。 |
| IS (Intention Shared)      | **意向共享锁**。事务打算在某个表的某些行上加 S 锁，提前在表级做标记，不会阻塞行锁。              |
| IX (Intention Exclusive)   | **意向排他锁**。事务打算在某个表的某些行上加 X 锁。                                          |
| S,GAP                      | **共享间隙锁**。锁定一个范围（gap），防止插入；多个事务可共享。                               |
| X,GAP                      | **排他间隙锁**。锁定一个范围，仅允许当前事务访问；用于避免幻读。                              |
| IS,GAP、IX,GAP             | **意向锁与间隙锁结合**，用于更复杂的间隙场景。                                                |
| X_REC_NOT_GAP              | **记录锁**。锁定确切的一条记录，不影响间隙（极少见，某些 `INSERT IGNORE` 场景会用到）。        |
| AUTO_INC                   | **自增锁**。用于锁住 `AUTO_INCREMENT` 的计数器，防止并发冲突。表级锁。                        |

