# Redis：中英文要点

用法：每个点先看中文理解，再背英文说法。回答公式：**原理 → 保证与取舍 → 你的项目例子**。标 [项目] 的地方可以结合你的 FXCenter 缓存、短链接设计和支付幂等来讲；标 [待补] 的是需要你补充真实情况的地方，不要编造。

## 1. Redis 是什么，为什么快

- **基于内存的键值数据库**，支持丰富的数据结构，常用作缓存、分布式锁、计数器、排行榜、简单队列。
- **为什么快：**
  1. 数据在**内存**中，读写是微秒级。
  2. 命令执行是**单线程**，没有锁竞争和线程切换。
  3. 使用**I/O 多路复用**（epoll），一个线程能处理大量连接。
  4. 数据结构本身高效（如跳表、压缩列表）。
- **单线程的准确说法：** 命令执行是单线程；Redis 6.0 之后网络读写可以用多线程，但**命令仍然由单线程执行**。持久化等后台工作由子进程或后台线程完成。
- **单线程的代价：** 一个慢命令会阻塞所有请求，所以要避免 `KEYS *`、对大集合做全量操作、超大 value。

**English:** "Redis is an in-memory key-value store. It's fast because data lives in memory, commands run on a single thread without lock contention, and it uses I/O multiplexing to handle many connections. The trade-off is that one slow command blocks everyone, so I avoid commands like KEYS and very large values."

## 2. 数据结构与使用场景

| 类型 | 典型场景 |
| --- | --- |
| **String** | 缓存对象（JSON）、计数器（`INCR`）、分布式锁（`SET key value NX PX`） |
| **Hash** | 存对象的多个字段，可以只改一个字段 |
| **List** | 简单队列、最新消息列表 |
| **Set** | 去重、共同好友、抽奖 |
| **Sorted Set (ZSet)** | 排行榜、延时队列（按时间戳排序）、限流（滑动窗口） |
| **Bitmap** | 签到、大量布尔状态；**Bloom filter 可以基于它实现** |
| **HyperLogLog** | 近似去重计数（如 UV），占用极小，有约 0.81% 误差 |
| **Stream** | 轻量级消息流，支持消费者组 |

**English:** "String for caching and counters, Hash for objects, Set for deduplication, Sorted Set for leaderboards and sliding-window rate limits, and Bitmap for compact boolean flags."

[项目] 短链接设计：Redis 做 cache-aside 缓存；Bloom filter 可以用 Bitmap（`SETBIT/GETBIT`）或 RedisBloom 模块。限流设计：用 Sorted Set 做滑动窗口，或 String 的 `INCR` + 过期做固定窗口。

## 3. 持久化：RDB 和 AOF

| | RDB | AOF |
| --- | --- | --- |
| 方式 | 某个时间点的**内存快照** | 记录**每条写命令**（追加日志） |
| 优点 | 文件小，恢复快 | 数据更完整，丢得少 |
| 缺点 | 两次快照之间的数据会**丢** | 文件大，恢复慢 |
| 丢数据 | 可能丢几分钟 | `everysec` 最多丢约 1 秒；`always` 几乎不丢但慢 |

- **AOF 的三种刷盘策略：** `always`（每条命令）、`everysec`（每秒，**常用折中**）、`no`（交给操作系统）。
- **AOF 重写：** 文件会越来越大，Redis 会在后台把它压缩成"生成当前数据所需的最少命令"。
- **RDB 和 AOF 怎么生成不阻塞主线程：** `fork` 出子进程，利用**写时复制（Copy-On-Write）**。fork 时内存很大会有短暂停顿。
- **常见做法：** 两个都开（混合持久化），重启时优先用 AOF 恢复。
- **要点：** Redis **作为缓存**时，丢数据通常可以接受（数据可以从 DB 重建）；**作为唯一数据源**时才需要认真考虑持久化。

**English:** "RDB takes point-in-time snapshots, so it's compact and fast to restore, but can lose recent writes. AOF logs every write, so it loses less, usually at most one second with everysec, but the file is larger. Many setups enable both. When Redis is only a cache, losing data is usually acceptable because we can rebuild it from the database."

## 4. 缓存穿透、击穿、雪崩（高频）

| | 现象 | 解决 |
| --- | --- | --- |
| **穿透 (penetration)** | 查询**根本不存在**的数据，缓存永远不命中，请求全部打到 DB | **Bloom filter**、缓存空值（短 TTL）、参数校验、限流 |
| **击穿 (breakdown)** | 一个**热点 key 过期**的瞬间，大量请求同时打到 DB | **互斥锁**（只让一个请求重建缓存）、热点数据**逻辑过期**或不过期、提前刷新 |
| **雪崩 (avalanche)** | **大量 key 同时过期**，或 Redis 宕机，DB 被压垮 | 过期时间加**随机值**、多级缓存、Redis 高可用、**限流降级**、预热 |

**记忆：** 穿透 = 查不存在的；击穿 = 一个热点失效；雪崩 = 一大片失效或整个挂掉。

**English:**
- "Cache penetration: requests for data that doesn't exist bypass the cache and hit the database. I use a Bloom filter, or cache the empty result with a short TTL."
- "Cache breakdown: a hot key expires and many requests hit the database at once. I use a mutex so only one request rebuilds the cache, or keep hot keys from expiring."
- "Cache avalanche: many keys expire together, or Redis goes down. I add random jitter to TTLs, make Redis highly available, and use rate limiting and degradation to protect the database."

[项目] 短链接里 Bloom filter → Redis → DB 解决的就是**穿透**；热点短链接用本地缓存（Caffeine）减轻单个 Redis 节点压力。

## 5. 缓存和数据库的一致性

- **最常用：Cache-Aside（旁路缓存）**
  - **读：** 先读缓存，没有则读 DB，再写入缓存。
  - **写：** **先更新 DB，再删除缓存**（不是更新缓存）。
- **为什么是"删除"而不是"更新"缓存：** 更新缓存在并发时容易写入旧值，而且有些缓存值计算成本高，删除后下次读时再算更简单（懒加载）。
- **为什么先更新 DB 再删缓存，而不是先删缓存：**
  - 先删缓存：线程 A 删缓存 → 线程 B 读到缓存没有，从 DB 读旧值并写回缓存 → 线程 A 才更新 DB。结果缓存里是**旧值**，且会一直留着。
  - 先更新 DB 再删缓存：出问题的概率小得多（需要"读 DB 在写 DB 之前，写缓存在删缓存之后"这种罕见时序），**但仍然不是绝对安全**。
- **进一步保证：**
  1. 给缓存设**过期时间**兜底，最终一致。
  2. **删除失败重试**：删除失败就放消息队列重试。
  3. **延迟双删**：更新 DB 后删一次，隔一小段时间再删一次，清掉并发读写回的旧值。
  4. **订阅 binlog（如 Canal）**，由独立服务根据数据库变更来删除缓存，业务代码不用管。
- **结论：** 缓存和 DB 是**最终一致**，不是强一致。需要强一致的数据（如账户余额）**不要依赖缓存**，直接读 DB。

**English:** "I use cache-aside: on read, check the cache and load from the database on a miss; on write, update the database first and then delete the cache. Deleting is safer than updating, because concurrent updates can write stale values. It's still only eventually consistent, so I add a TTL as a safety net and retry failed deletions. For data that needs strong consistency, like an account balance, I read from the database."

[项目] FXCenter 的做法：本地缓存是一个不可变快照，通过 volatile 引用整体替换，且只在完整性检查通过后才切换，所以读线程要么看到完整的旧快照，要么看到完整的新快照。[待补] 你们刷新快照的触发方式（定时拉取或事件通知）。

## 6. 分布式锁

- **基本实现：**
  ```
  SET lock:order:123 <唯一值> NX PX 30000
  ```
  - `NX`：key 不存在时才设置，**保证只有一个客户端能拿到锁**（原子操作）
  - `PX 30000`：30 秒后自动过期，**防止持有者崩溃后锁永远不释放**
  - 值用**唯一标识**（如 UUID），释放时要**确认是自己的锁**
- **释放锁：** 必须是"判断是自己的锁 + 删除"的**原子操作**，用 Lua 脚本：
  ```lua
  if redis.call('get', KEYS[1]) == ARGV[1] then
      return redis.call('del', KEYS[1])
  else
      return 0
  end
  ```
  否则可能删掉别人的锁（自己的锁已过期，别人拿到了，自己再去删）。
- **锁过期但业务没执行完怎么办：**
  - 设合理的过期时间；
  - **看门狗（watchdog）续期**：Redisson 会在持有锁期间自动延长过期时间；
  - 更重要的是：业务本身要**幂等**，并用**fencing token** 或数据库的**版本号、唯一约束**做最终保护。
- **可重入、公平：** Redisson 提供可重入锁、读写锁、公平锁等。
- **Redis 分布式锁不是绝对安全：**
  1. 主从切换时，主节点还没把锁同步给从节点就宕机，**新主上没有这把锁**，两个客户端同时持有。
  2. 持有者因 **GC 停顿或网络延迟**导致锁过期，自己还以为持有锁。
  - **Redlock** 算法用多个独立节点来缓解，但争议较大。
  - **务必说出：** 锁只是**效率优化**，**正确性**要靠数据库的唯一约束、乐观锁或条件更新来保证。
- **和 ZooKeeper / etcd 比：** 后两者用强一致协议和临时节点（或租约），更安全但更慢、更重。

**English:** "I acquire the lock with SET key unique-value NX PX, so only one client succeeds and the lock expires if the holder crashes. I release it with a Lua script that checks the value before deleting, so I never delete someone else's lock. A watchdog can extend the lock while the work is running. But a Redis lock isn't fully safe, because of failover and long pauses, so I never rely on it alone for correctness. I back it with idempotency and a database constraint or version check."

[项目] 和支付设计呼应：恢复任务用**租约**认领记录，就是"带过期时间的锁"；最终保证来自**条件更新**，而不是锁本身。TasCreed 任务抢占也是同类思路。

## 7. 高可用：主从、哨兵、Cluster

| 方案 | 作用 | 说明 |
| --- | --- | --- |
| **主从复制** | 读写分离、数据备份 | 主节点写，从节点异步复制，**可能丢最近的写** |
| **哨兵 (Sentinel)** | **自动故障转移** | 监控主节点，主节点挂了就选一个从节点升为主；通知客户端 |
| **Cluster（集群）** | **数据分片 + 高可用** | 数据分成 **16384 个哈希槽**，分布在多个主节点，每个主节点有从节点 |

- **Cluster 怎么分片：** `CRC16(key) % 16384` 得到槽，槽分配给节点。扩缩容时**迁移槽**，不用移动全部数据（和一致性哈希、逻辑分片是同一类思路）。
- **Cluster 的限制：** 多 key 操作（如事务、Lua）要求 key 在**同一个槽**，可以用 **hash tag**（`{user1}:a` 和 `{user1}:b` 用花括号内的部分计算槽）。
- **主从复制是异步的**，所以故障转移时可能丢数据，这也是分布式锁不绝对安全的原因之一。
- **脑裂：** 网络分区时出现两个主节点，旧主上的写入会在恢复后丢失。可以配置 `min-replicas-to-write` 缓解。

**English:** "Replication gives backups and read scaling, but it's asynchronous. Sentinel monitors the master and promotes a replica on failure. Redis Cluster shards data across 16,384 hash slots over multiple masters, each with replicas, so it scales both capacity and throughput."

## 8. 过期删除与内存淘汰

- **过期 key 怎么删（两种结合）：**
  1. **惰性删除：** 访问到这个 key 时才检查，过期就删。
  2. **定期删除：** 周期性随机抽查一批 key，删掉过期的。
  - 所以过期的 key **不一定立刻被释放内存**。
- **内存满了怎么办（`maxmemory-policy`）：**
  | 策略 | 说明 |
  | --- | --- |
  | `noeviction` | 不淘汰，写入报错（默认） |
  | `allkeys-lru` | 所有 key 中淘汰最久未使用的，**做缓存时常用** |
  | `allkeys-lfu` | 淘汰访问频率最低的 |
  | `volatile-lru` / `volatile-ttl` | 只在设置了过期时间的 key 中淘汰 |
  | `allkeys-random` | 随机淘汰 |
- Redis 的 LRU 是**近似 LRU**（随机采样一部分 key，淘汰其中最久未用的），不是严格的双向链表。

**English:** "Expired keys are removed lazily on access and by periodic sampling. When memory is full, the eviction policy decides what to drop; for a cache I usually use allkeys-lru."

## 9. 其他常见问题

- **大 key（big key）：** 单个 value 很大（如巨大的 Hash、List）。问题：读写慢、阻塞单线程、网络拥塞、删除时卡顿。处理：拆分、用 `UNLINK` 异步删除、避免全量读取。
- **热 key：** 某个 key 访问量极高，压在一个节点上。处理：**本地缓存**、多副本分散、拆成多个 key。
- **Pipeline：** 一次发送多条命令，减少网络往返，但**不保证原子性**。
- **事务：** `MULTI/EXEC` 把命令排队后依次执行，**不支持回滚**；`WATCH` 实现乐观锁。需要原子地做"检查再操作"时，用 **Lua 脚本**（Redis 保证脚本执行期间不被打断）。
- **Redis 和 Memcached：** Redis 数据结构丰富、支持持久化和高可用；Memcached 只有简单 key-value，多线程。
- **Redis 能做消息队列吗：** List、Stream 可以做轻量队列，但可靠性、堆积能力和生态不如 Kafka，关键业务用专门的 MQ。

**English:** "For an atomic check-then-act, I use a Lua script, because Redis runs it without interruption. Pipelines reduce round trips but aren't atomic, and MULTI/EXEC doesn't roll back."

## 10. 30 秒回答

"Redis 是内存键值库，快是因为内存、单线程命令执行和 I/O 多路复用。我主要用它做缓存，采用 cache-aside：读时缓存未命中就查 DB 并回填，写时先更新 DB 再删缓存，再用 TTL 兜底，保证最终一致。要防三类问题：穿透用 Bloom filter 或缓存空值，击穿用互斥锁或热点不过期，雪崩用过期时间加随机值和限流降级。分布式锁用 `SET NX PX` 加 Lua 释放，必要时看门狗续期，但它不是绝对安全，正确性靠幂等和 DB 约束。高可用用主从加哨兵或 Cluster。"

## 11. 追问清单（自测）

1. Redis 为什么快？单线程为什么还这么快？Redis 6 的多线程做了什么？
2. 穿透、击穿、雪崩分别是什么？怎么解决？
3. 为什么写操作是"先更新 DB，再删缓存"？还有什么情况会不一致？怎么兜底？
4. 怎么用 Redis 实现分布式锁？为什么释放锁要用 Lua？锁过期了怎么办？
5. Redis 分布式锁为什么不绝对安全？你怎么保证正确性？
6. RDB 和 AOF 的区别？线上怎么选？
7. 主从、哨兵、Cluster 分别解决什么问题？Cluster 怎么分片？
8. 过期 key 怎么删？内存满了怎么淘汰？
9. 什么是大 key、热 key？怎么处理？
10. 怎么原子地完成"检查再操作"？（Lua、WATCH、`SET NX`）

## 12. 待补（需要你补充真实情况，不要编造）

- [待补] 你们项目里 Redis 的实际使用场景（缓存什么、TTL 多久、一致性怎么处理）。
- [待补] FXCenter 缓存快照的刷新触发方式。
- [待补] 是否在项目里用过分布式锁，用的哪种实现。
