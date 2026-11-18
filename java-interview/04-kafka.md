# Kafka：中英文要点

用法：每个点先看中文理解，再背英文说法。回答公式：**原理 → 保证与取舍 → 你的项目例子**。你的简历里有 "消费 Kafka 支付记账数据，持久化到 HDFS" 的 ETL，面试官很可能顺着这条问，所以每节都标了 [项目]。

## 1. Kafka 是什么，为什么用

- **分布式、高吞吐、持久化的日志（commit log）系统**，常用作消息队列和事件流平台。
- **三个典型用途：**
  1. **异步解耦**：生产者不用等消费者处理完。
  2. **削峰填谷**：流量高峰时消息先堆在 Kafka，消费者按自己的速度处理。
  3. **数据管道**：把数据从一个系统传到另一个（如业务系统 → HDFS、Spark）。
- **和传统 MQ 的区别：** 消息**不会因为被消费就删除**，按保留时间或大小清理；多个消费者组可以**各自独立地重复读取**同一份数据。

**English:** "Kafka is a distributed, durable commit log. I use it to decouple services, absorb traffic spikes, and move data between systems. Unlike a traditional queue, messages aren't deleted when consumed, so multiple consumer groups can read the same data independently."

[项目] 支付记账数据通过 Kafka 流入 ETL，再写到 HDFS：生产方和下游处理解耦，ETL 慢了数据也不会丢。

## 2. 核心概念

| 概念 | 说明 |
| --- | --- |
| **Broker** | Kafka 服务节点，一组 broker 构成集群 |
| **Topic** | 消息的逻辑分类 |
| **Partition（分区）** | Topic 被分成多个分区，每个分区是一个**有序的、只追加的日志**。并行度和扩展性的基础 |
| **Offset** | 消息在分区内的**唯一序号**，只在分区内有意义 |
| **Producer** | 生产者，把消息发到 topic |
| **Consumer** | 消费者，从 topic 拉取消息（pull 模式） |
| **Consumer Group** | 一组消费者共同消费一个 topic，**组内每个分区只被一个消费者消费** |
| **Replica（副本）** | 每个分区有多个副本，分布在不同 broker |
| **Leader / Follower** | 每个分区有一个 leader 负责读写，follower 从 leader 同步 |
| **ISR（In-Sync Replicas）** | 和 leader 保持同步的副本集合 |

**English:** "A topic is split into partitions, and each partition is an ordered, append-only log. A message's position in a partition is its offset. A consumer group shares the work: each partition is consumed by only one consumer in the group."

## 3. 消息路由与有序性

- **生产者怎么选分区：**
  - 指定了 key → `hash(key) % 分区数`，**相同 key 进入同一个分区**
  - 没指定 key → 轮询（或粘性分区策略）
- **有序性：只保证分区内有序，不保证 topic 全局有序。**
- **需要顺序怎么办：** 用**业务 key**（如订单 id、账户 id）作为消息 key，同一个 key 的消息在同一个分区，就能按顺序消费。
- **注意：** 增加分区数会改变 `hash(key) % 分区数` 的结果，**同一个 key 会换分区**，顺序可能被破坏，所以分区数要提前规划。

**English:** "Kafka only guarantees order within a partition. To keep related messages in order, I use a business key, such as the account id, as the message key, so they go to the same partition."

[项目] 记账数据如果要求同一账户的流水有序，就应该用账户 id 作为 key。[待补] 你们实际的 key 和分区策略。

## 4. 消费者组与 Rebalance

- **组内每个分区只能被一个消费者消费**，所以**消费者数量超过分区数，多出来的消费者会闲置**。并行度上限 = 分区数。
- **Rebalance（重平衡）：** 消费者加入或退出、分区数变化时，重新分配分区。
  - **代价：** 重平衡期间，整个组暂停消费（Stop-The-World），可能造成积压和重复消费。
  - **常见触发原因：**
    1. 消费者实例启停（发布、扩缩容）
    2. **心跳超时**（`session.timeout.ms`）
    3. **处理时间过长**，超过 `max.poll.interval.ms`，被认为挂了
  - **减少 Rebalance：** 单次处理不要太久、调小 `max.poll.records`、合理设置超时、使用静态成员（`group.instance.id`）和增量协作式重平衡（Cooperative）。

**English:** "In a consumer group, each partition goes to exactly one consumer, so the parallelism is limited by the number of partitions. A rebalance reassigns partitions when consumers join or leave, and it pauses consumption. Slow processing can trigger it, because the consumer exceeds max.poll.interval.ms and is considered dead."

## 5. Offset 提交

- 消费者要**记录自己消费到哪**，存在 Kafka 的内部 topic `__consumer_offsets` 中。
- **自动提交**（`enable.auto.commit=true`）：定时提交。简单，但**可能丢消息或重复消费**。
- **手动提交**（推荐）：处理完业务后再提交，`commitSync()` 或 `commitAsync()`。

**两种顺序的后果：**
| 做法 | 崩溃时的后果 | 语义 |
| --- | --- | --- |
| **先提交 offset，再处理** | 处理前崩溃 → 消息丢失 | at-most-once |
| **先处理，再提交 offset** | 处理后、提交前崩溃 → 重启后**重复处理** | **at-least-once（最常用）** |

**English:** "If I commit the offset before processing, a crash loses the message, which is at-most-once. If I process first and commit afterward, a crash causes reprocessing, which is at-least-once. I choose at-least-once and make the processing idempotent."

## 6. 消息不丢失（高频）

消息链路分三段，每一段都要保证：

**① 生产者端：**
- `acks=all`（或 -1）：**所有 ISR 副本**都写入才算成功。`acks=1` 只要 leader 写入，leader 挂了可能丢；`acks=0` 不等确认，最不可靠。
- **开启重试**：`retries` 设大，配合 `delivery.timeout.ms`。
- 使用**带回调的发送**，处理失败（日志、告警、落库重发），不要只"发出去就算"。

**② Broker 端：**
- **副本数 ≥ 3**：`replication.factor=3`
- **`min.insync.replicas=2`**：ISR 里至少有 2 个副本在线，否则写入失败（配合 `acks=all`）
- **关闭不干净的 leader 选举**：`unclean.leader.election.enable=false`，不让落后太多的副本当 leader（否则会丢数据）
- 刷盘依赖多副本，而不是靠每条同步刷盘。

**③ 消费者端：**
- **手动提交 offset，并且在业务处理完成之后再提交。**

**English:** "To avoid losing messages, on the producer I use acks=all with retries and handle send failures. On the broker, I use a replication factor of three, min.insync.replicas of two, and disable unclean leader election. On the consumer, I commit offsets manually, after processing."

## 7. 重复消费与幂等（高频）

**重复来源：**
- 生产者重试（超时但其实已写入）
- 消费者先处理后提交，崩溃重启
- Rebalance 导致未提交的消息被重新分配

**解决：**
1. **生产者幂等：** `enable.idempotence=true`，Kafka 用 **producer id + 序列号**去重，保证单个分区、单次会话内不重复。
2. **消费者幂等（最关键）：** 业务层去重。
   - **唯一键约束**：数据库用业务 id 或消息 id 建唯一索引，重复插入失败就忽略
   - **状态机检查**：只在状态符合时更新
   - **幂等写入**：固定的输出文件名，重跑覆盖
   - **`INSERT ... ON DUPLICATE KEY UPDATE`**
3. **事务（Exactly-once）：** Kafka 支持事务（`transactional.id`），在 **Kafka 内部的"读 - 处理 - 写"** 流程中保证恰好一次（EOS）。但**涉及外部系统（数据库、HDFS）时，仍需要消费者幂等**。

**English:** "Kafka gives at-least-once by default, so duplicates can happen from producer retries or consumer restarts. On the producer, I enable idempotence. The most important part is the consumer: I make processing idempotent, for example with a unique key on the message id. Kafka's transactions give exactly-once only inside Kafka. With external systems, I still need idempotent writes."

[项目] 和 TasCreed 的思路一致：**at-least-once + 幂等**。ETL 写 Parquet 到 HDFS，可以通过固定文件名或按分区 + offset 范围命名，重跑覆盖，不产生重复。[待补] 你们实际的幂等做法。

## 8. 消息积压（高频场景题）

**先判断原因：** 是**生产变快了**，还是**消费变慢了**？看 consumer lag（消费滞后量）。

**排查和处理步骤：**
1. **看消费者是否有问题：** 是否报错一直重试、是否在频繁 Rebalance、处理逻辑是否变慢（慢 SQL、下游接口慢）。
2. **临时扩容：**
   - 增加消费者实例，**不超过分区数**才有效。
   - 如果分区数不够：新建一个更多分区的 topic，写临时程序把积压消息快速转发过去，用更多消费者处理。
3. **提高单个消费者的吞吐：**
   - 消费者内部**批量处理**（批量写库）、用线程池并行处理（注意 offset 提交和顺序问题）
   - 调大 `max.poll.records`、`fetch.min.bytes` 等
4. **长期方案：**
   - 合理规划分区数
   - 慢的下游做限流、降级、异步化
   - 积压监控和告警（lag 阈值）
5. **注意：** 如果消息有保留期限，积压超过保留时间，消息会被**删除**，造成丢失，所以要在到期之前处理完。

**English:** "First I check whether production increased or consumption slowed down by looking at consumer lag. Then I check for errors, rebalances, or slow downstream calls. To scale out, I add consumers, but only up to the number of partitions. If partitions aren't enough, I forward the backlog into a new topic with more partitions. I also process in batches and set lag alerts."

## 9. Kafka 为什么快（高吞吐）

1. **顺序写磁盘**：日志只追加，顺序写比随机写快得多，接近内存速度。
2. **页缓存（Page Cache）**：依赖操作系统缓存，读写先走内存。
3. **零拷贝（Zero-copy）**：用 `sendfile`，数据从磁盘页缓存直接发到网卡，不经过用户态，省去多次拷贝。
4. **批量处理和压缩**：生产者批量发送，消息可压缩（gzip、snappy、lz4、zstd），减少网络和磁盘开销。
5. **分区并行**：多个分区分布在多个 broker，并行读写。

**English:** "Kafka is fast because it writes sequentially to disk, relies on the OS page cache, uses zero-copy to send data from the page cache to the network, batches and compresses messages, and spreads load across partitions."

## 10. 副本与高可用

- 每个分区有 **1 个 leader + N 个 follower**，读写只走 leader，follower 同步数据。
- **ISR：** 和 leader 保持同步的副本集合。follower 落后太久（`replica.lag.time.max.ms`）会被踢出 ISR。
- **leader 挂了：** 从 ISR 中选新的 leader，保证数据不丢。
- **Controller：** 集群中有一个 controller broker 负责选举 leader、管理分区状态。早期靠 ZooKeeper，**新版本（KRaft）已不依赖 ZooKeeper**。
- **高水位（HW, High Watermark）：** 消费者只能读到 HW 之前的、已被所有 ISR 副本确认的消息。

**English:** "Each partition has a leader and followers. Only the leader serves reads and writes. If the leader fails, a new leader is elected from the in-sync replicas, so no committed data is lost."

## 11. 其他常见问题

- **Pull 还是 Push？** 消费者主动 **pull**。好处：消费者按自己的速度消费，不会被压垮；代价：没有消息时需要轮询（用长轮询缓解）。
- **消息保留：** 按时间（默认 7 天）或大小；也可以用**日志压缩（log compaction）**，只保留每个 key 的最新值。
- **消息有序 + 高吞吐的矛盾：** 单分区有序但并行度低。取舍是用业务 key 分区，**同一个 key 有序，不同 key 并行**。
- **死信队列（DLQ）：** 反复处理失败的消息发到专门的 topic，不阻塞正常消息，方便后续人工处理。（和 TasCreed 的"无限重试的任务用 kill/ban 处理"是同类问题。）
- **Kafka 和 RabbitMQ 怎么选：** Kafka 适合高吞吐、日志和流式数据、可重放；RabbitMQ 适合复杂路由、低延迟、任务队列场景。

## 12. 面试高频问题清单（自测）

1. Kafka 的核心概念：topic、partition、offset、consumer group？
2. 怎么保证消息有序？
3. 怎么保证消息不丢失？（生产者、broker、消费者三段）
4. 怎么避免重复消费？怎么做幂等？
5. at-most-once、at-least-once、exactly-once 有什么区别？
6. 什么是 Rebalance？为什么会发生？怎么减少？
7. 消息积压了怎么处理？
8. Kafka 为什么快？
9. 消费者数量比分区多会怎样？
10. ISR 是什么？leader 挂了怎么办？
11. offset 什么时候提交？先提交还是先处理？
12. 如果消息一直处理失败怎么办？

## 13. 和你项目的连接

| 知识点 | 你的故事 |
| --- | --- |
| 消费 Kafka 数据写入 HDFS | 支付记账 ETL 管道：Kafka → Parquet → HDFS → Hive/Spark |
| at-least-once + 幂等 | ETL 重跑不产生重复；和 TasCreed 的"固定文件名，重跑覆盖"同一思想 |
| 合并 Parquet 小文件、对账 | 批处理定期合并并校验数据，作为数据完整性的兜底 |
| 积压与节点资源 | TasCreed：节点过载后 kill、调整并发参数 |
| 无限失败的消息 | 类似"无限 retry 的任务"：重试要有上限，之后转死信队列或人工处理 |

[待补] 以下请按真实情况填写，没有就不要编：
- 你们的 topic、分区数、消息 key、消费者组怎么配置
- offset 提交方式、重复数据的处理方式
- 遇到过的积压或重平衡问题，怎么发现和解决
- 数据量级和延迟要求
