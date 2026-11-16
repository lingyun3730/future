# Distributed Configuration Store – System Design 准备

题目：设计一个分布式配置中心（类似 etcd、ZooKeeper、Consul、Apollo）。应用从中读取配置、开关、参数，修改后不用重新发布代码。在支付公司里，配置错误可能直接导致线上故障，所以重点是**正确、安全地改配置**。

## One-liner

> "A configuration store keeps configs consistent, versioned and safe to change. Writes go through a strongly consistent core like Raft. Clients cache configs and get updates by watch, so they keep working if the store is down. Every change is versioned, reviewed, rolled out gradually, and can be rolled back quickly."

## 1. Clarify

| 问题 | 为什么问 |
| --- | --- |
| 存什么？小的 key-value、JSON、还是大文件？ | 决定大小限制，配置一般很小（KB） |
| 读写比例和规模？ | 读远多于写：多少服务、多少实例、多少 key |
| 一致性要求？ | 改完所有实例多久生效？是否允许短时间不一致？ |
| 谁能改？要不要审批？ | 权限和审计，支付场景通常要审批 |
| 要不要灰度、回滚？ | 配置错误是故障主因，通常需要 |
| 配置中心挂了怎么办？ | 应用不能因此挂，需要本地缓存 |
| 多环境、多 region？ | dev / staging / prod 隔离；跨 region 同步 |
| 有无敏感配置？ | 密钥放专用 secret 管理，不放普通配置里 |

## 2. 核心设计

```text
管理端（UI / API，权限 + 审批）
      |
   Config Service（校验、版本、审计）
      |
   强一致存储（Raft：etcd / 自建）
      |  watch / 轮询
客户端 SDK（本地缓存 + 本地文件快照）---> 应用
```

| 组件 | 职责 |
| --- | --- |
| 强一致存储 | 保存配置和版本，Raft 保证多数派写入后才成功，所有节点看到同一份 |
| Config Service | 校验配置格式、权限检查、写入新版本、记录审计日志、触发发布 |
| 客户端 SDK | 启动时拉取全量配置，之后用 watch（长连接推送）或定期轮询获取变更；缓存在内存，并落本地文件 |
| 管理端 | 提交、审批、发布、回滚、查看历史 |

## 3. 关键要点

### 一致性：为什么用强一致的存储

配置是系统的"事实来源"，多个节点必须看到同一份，否则不同实例用着不同规则（例如一半实例用旧的汇率规则）。所以写入走 Raft，多数派确认后才算成功。你在 TasCreed 用 etcd，原因同理：强一致才能保证同一个任务只有一个 worker 抢到。

**读的一致性：** 大多数应用读本地缓存，允许短暂延迟（最终一致）；需要强读的场景（例如发布流程自己）才直接读存储的 leader。

### 版本管理与回滚

- 每次修改产生新版本号（不可变），保留完整历史。
- 回滚就是把当前指针切回旧版本，是一次普通的发布，快且安全。
- 追加式历史，不改旧版本（和你 EasyFin 的 history 表同理）。

### 灰度发布

```text
新版本 -> 先对 1% 实例 / 某个环境生效 -> 观察指标（错误率、成功率）
  -> 没问题扩大到 10% -> 50% -> 100%
  -> 有问题自动或手动回滚
```

- 按实例、环境、region、租户分批。
- 发布和监控联动：指标恶化自动停止。

### 推送与缓存

| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| Watch / 长连接推送 | 生效快 | 连接数多，需要处理断线重连 |
| 定期轮询（带版本号） | 简单可靠 | 有延迟，请求量大 |
| 组合 | watch 为主，轮询兜底 | 实现稍复杂 |

- 客户端本地缓存加本地文件快照：配置中心不可用时，应用用最后一次成功的配置继续运行，**配置中心不能成为单点故障**。
- 更新应原子：配置对象整体替换，不能让应用读到半新半旧的配置。

### 校验与安全

- 写入前校验：schema、取值范围、引用是否存在（EasyFin 做的 validation 就是这个）。
- 权限：按环境、按 key 前缀授权；生产环境修改要审批（两人原则）。
- 审计：谁、何时、改了什么、为什么，保留完整日志。
- 敏感信息（密钥、密码）不放普通配置，用专用的 secret 管理，加密存储。

### 故障与可观测

| 故障 | 处理 |
| --- | --- |
| 配置中心节点挂了 | Raft 多数派存活即可继续；少数节点故障不影响 |
| 整个配置中心不可用 | 客户端用本地缓存继续运行，只是不能改配置 |
| 错误配置上线 | 灰度发现、快速回滚；变更必须关联监控 |
| 客户端长期不更新 | 监控每个实例当前的配置版本，落后的告警 |
| 网络分区 | 少数派一侧不能写；客户端用缓存，恢复后同步 |

监控：各实例生效的版本、发布进度、拉取失败率、写入延迟、审批和变更记录。

## 4. 追问与答案

| 问题 | 答案要点 |
| --- | --- |
| Why strong consistency for configs? | 避免不同实例同时使用不同规则；写入由 Raft 多数派确认 |
| Consistency vs availability? | 写路径选一致性（少数派不能写）；读路径选可用性（客户端缓存，允许短暂旧值） |
| What if the config store is down? | 应用用本地缓存和文件快照继续运行；不能改配置，但不影响现有服务 |
| How do you avoid a bad config taking everything down? | 写前校验、审批、灰度发布、指标联动、快速回滚 |
| How do you roll back? | 版本不可变，切回旧版本指针即可，作为一次新的发布 |
| Push or pull? | watch 推送为主，带版本号的轮询兜底 |
| How do clients get an atomic update? | 整体替换配置对象，不原地修改 |
| How do you scale reads? | 客户端缓存；中间加只读代理或缓存层；存储只处理写和少量读 |
| How do you handle secrets? | 不放在普通配置里，用专用 secret 管理，加密、访问控制和审计 |
| Multi-region? | 每个 region 有本地读副本，写入走主 region；跨 region 异步同步，接受短暂延迟 |
| Config as code or as data? | 规则类配置（如会计映射）是数据，可在 UI 里改；影响大的要审批和校验 |

## 5. 你的经验怎么接

| 经历 | 怎么接 |
| --- | --- |
| **EasyFin** | 本质是"配置数据的提交、校验、审批、history、集成"：Excel 导入 → validation → recon（与 DB 一致）→ 审批 → 生成 SQL → 工程审批 → staging。对应本题的校验、版本历史、审批、分阶段发布 |
| **TasCreed + etcd** | 用 etcd 做强一致协调：lease 绑定 key、watch、原子写入 |
| **FXCenter** | 汇率和规则的缓存与 fallback：客户端本地有缓存，缺失时整批 fallback，并通知 recon |
| **Accounting system** | 记账规则以配置数据存在 DB 中，变更影响分录，所以需要严格的校验和审批 |

注意：具体说到的实现细节以真实经历为准，不确定的部分要说明，不要说成已确认事实。

## 6. 一分钟总结稿

> "I would design the config store around safe changes. Configs are small key-values, stored in a strongly consistent system like etcd, so every node sees the same version. Each change creates a new immutable version, so we can see the history and roll back quickly. Changes are validated and reviewed before they're applied, and rolled out gradually with metrics watching, because a bad config can take down production. Clients cache configs locally and update by watch, so if the store is down, the applications keep running with the last good config. Everything is audited, and secrets are kept in a separate secret manager."
