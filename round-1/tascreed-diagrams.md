# TasCreed 架构与流程图

基于开源仓库 `TasCreed` 的文档和代码整理（`TaskOccupyHelper`、`TaskHeartBeatThread`、`BaseOccupyEtcdBulletin`、`JobWatcherRoutineExecutor`、`States.md`）。
图使用 mermaid 编写，GitHub / VS Code（Markdown Preview Mermaid 插件）可直接渲染。

> 内部线上版本与开源版本的差异待补，下面的图只代表开源版本。

---

## 1. 整体架构

每个 App 节点功能对等（无 leader），TasCreed 以 library 形式嵌入业务应用。

```mermaid
flowchart LR
    user(User / Scheduler)

    subgraph N1[App Node 1]
        direction TB
        S1[Job Server<br/>REST API]
        W1[Task Worker<br/>Watcher + Worker + Heartbeat]
        R1[Routine Executor<br/>job-watcher 等]
    end

    subgraph N2[App Node 2]
        direction TB
        S2[Job Server]
        W2[Task Worker]
        R2[Routine Executor]
    end

    subgraph Storage
        bulletin[(Bulletin: etcd<br/>alive job / task<br/>adoption + lease)]
        archive[(Archive: ES / etcd)]
    end

    user -->|submit job request| S1
    S1 -->|create job| bulletin
    R1 -->|advance job, create tasks| bulletin
    R1 -->|archive SUCCESS job| archive
    W1 -->|occupy / heartbeat / update| bulletin
    W2 -->|occupy / heartbeat / update| bulletin
    R2 -->|occupy routine| bulletin
    S2 -->|create job| bulletin
```

## 2. 核心概念模型

```mermaid
erDiagram
    JobDefine ||--o{ StepDefine : has
    JobDefine ||--o{ Job : instantiates
    JobRequest ||--|| Job : triggers
    Job ||--o{ Step : has
    StepDefine ||--o{ Step : instantiates
    Step ||--|{ Task : "generates (sharding N)"

    Job {
        string jobName
        string trigger "e.g. 20191113"
        string state
    }
    Step {
        string stepName
        string state
        map taskStates
    }
    Task {
        string jobName
        string trigger
        string stepName
        int shardingIndex
        int shardingTotal
    }
```

## 3. Job 提交流程

```mermaid
sequenceDiagram
    actor U as User
    participant JS as Job Server
    participant B as Bulletin (etcd)
    participant A as Archive

    U->>JS: POST job request (jobName, trigger, params)
    JS->>JS: find JobDefine, validate, build Job
    JS->>B: check job exists?
    JS->>A: check job exists?
    alt found in bulletin or archive
        JS-->>U: reject duplicate
    else new job
        JS->>B: write new job
        JS-->>U: accepted
    end
```

## 4. Job 推进（job-watcher routine）

```mermaid
sequenceDiagram
    participant R as job-watcher routine<br/>(interval 30s, scale 3)
    participant JS as Job Server
    participant B as Bulletin
    participant A as Archive

    loop every 30s
        R->>JS: refreshAll()
        JS->>B: read alive jobs + done tasks
        JS->>JS: update step / job state by DAG
        JS->>B: create todo tasks for ready steps
        opt job SUCCESS
            JS->>A: archive job and tasks
            JS->>B: delete from bulletin
        end
    end
```

## 5. Task 抢占与执行（核心）

```mermaid
sequenceDiagram
    participant TW as Task Watcher
    participant B as Bulletin (etcd)
    participant WK as Task Worker thread
    participant HB as Heartbeat thread

    loop periodically
        TW->>B: acquire taskAdoptionLock
        TW->>B: read adoptions + todo tasks
        TW->>TW: adopted >= maxWorkerCountOverall ? stop
        TW->>TW: filter by ban / affinity / job ratio
        TW->>TW: pick max weight (job priority)
        TW->>B: grantLease(TTL) + put adoptionKey with lease
        TW->>B: release lock
    end
    TW->>WK: start worker with picked task
    WK->>HB: start heartbeat(occupyInfo)
    par execute
        WK->>WK: TaskExecutor.execute() (idempotent)
    and keep alive
        loop every TTL/3
            HB->>B: keepAliveOnce(leaseId)
        end
    end
    WK->>B: update task state (SUCCESS / FAILED / ERROR)
    WK->>B: revoke lease, remove adoption
    WK->>HB: stop
```

## 6. 节点宕机后的故障恢复

```mermaid
sequenceDiagram
    participant A as Node A (worker)
    participant B as etcd
    participant C as Node B (worker)

    A->>B: occupy task T with lease (TTL)
    A->>A: executing T
    Note over A: node crash / hang
    A--xB: heartbeat stops
    Note over B: lease expires after TTL
    B->>B: adoption key auto-deleted
    C->>B: read todo tasks + adoptions
    Note over C: T still todo and not occupied
    C->>B: occupy T with new lease
    C->>C: re-execute T
    Note over C: at-least-once, so executor must be idempotent
```

## 7. 线程与职责模型

```mermaid
flowchart TB
    subgraph Node[TasCreed Node]
        direction TB
        subgraph SERVER[SERVER duty]
            JS[JOB_SERVER<br/>job APIs + state refresh]
            SS[STATE_SERVER<br/>ban / delete APIs]
            SCH[SCHEDULE_SERVER<br/>schedule APIs]
        end
        subgraph EXECUTOR[EXECUTOR duty]
            TE[TASK_EXECUTOR<br/>TaskWatcher -> TaskWorker + Heartbeat]
            RE[ROUTINE_EXECUTOR<br/>RoutineWatcher -> Routine + Heartbeat]
        end
    end
    Rule[Node duty rule<br/>minValidTcVersion / hostName regex] -->|disable duties if invalid| Node
```

## 8. Step 状态机

```mermaid
stateDiagram-v2
    [*] --> DORMANT
    DORMANT --> START: dependencies done, start creating tasks
    DORMANT --> IGNORED: ignored by request
    DORMANT --> SKIP_BY_ERROR: dependent step ERROR
    DORMANT --> SKIP_BY_FAILED: dependent FAILED, cannot fail
    DORMANT --> ACCEPTABLE_FAILED: dependent FAILED, CAN_FAIL
    START --> READY: all tasks created
    START --> FAILED: task failed, cannot fail
    START --> ACCEPTABLE_FAILED: task failed, CAN_FAIL
    START --> ERROR: task error
    READY --> SUCCESS: all tasks SUCCESS
    READY --> FAILED: task failed, cannot fail
    READY --> ACCEPTABLE_FAILED: task failed, CAN_FAIL
    READY --> ERROR: task error
    SUCCESS --> [*]
    FAILED --> [*]
    ERROR --> [*]
```

## 9. Task / Job 状态

```mermaid
flowchart LR
    subgraph Task
        direction LR
        TU[UNDONE] -->|executor sets success| TS[SUCCESS]
        TU -->|executor sets failed, retryable| TF[FAILED]
        TU -->|unexpected exception| TE[ERROR]
    end
    subgraph Job
        direction LR
        JU[UNDONE] -->|all steps success| JS[SUCCESS -> archive]
        JU -->|any step FAILED| JF[FAILED, stay in bulletin]
        JU -->|any step ERROR| JE[ERROR, stay in bulletin]
        JU -->|circular dependency| JK[STUCK]
    end
```

## 10. 示例：报表 Job 的 Step DAG（示意）

示例结构来自 `docs/design/Concept.md` 的 `balance-report`，真实报表 Job 的 Step 与分片数待补。

```mermaid
flowchart LR
    A[generate_files_and_upload_by_sharding<br/>sharding = 12<br/>12 tasks in parallel] --> B[merge_sharding_files_and_upload<br/>1 task]
    B --> C[SUCCESS job -> archive]
```

---

## 11. 适用场景（应用角度）

以下根据框架设计推导：至少一次执行、Task 为分钟级粒度、Job Watcher 30 秒轮询、抢 Task 有全局锁。

### 适合

| 场景 | 为什么适合 | 用到的能力 |
| --- | --- | --- |
| 日终 / 定时批处理（报表、对账、结算） | 节点挂了无需人工介入，Job 最终会跑完；`jobName + trigger`（如日期）天然防重复提交 | Job 去重、故障自动恢复、Schedule 定时触发 |
| 大批量数据分片并行处理 | 一个 Step 拆成 N 个 Task，分散到多节点并行 | `sharding`、Step DAG（分片生成 → 合并上传） |
| 多阶段流水线（ETL、生成文件 → 校验 → 上传 → 通知） | Step 依赖构成 DAG，上游成功才会触发下游 | Step 依赖、`CAN_FAIL`、Phase |
| 长驻的小批量消费任务 | 需要集群内保证只有指定数量的实例在跑，挂了自动接管 | Routine（`scale`、`interval`） |
| 需要灰度或多机房的任务集群 | 按版本、主机名控制哪些节点可以执行 | Node Duty、Ban |
| 失败后需要人工重试的批任务 | 非 SUCCESS 的 Job 留在 bulletin，可以手动重试 | FAILED / ERROR 状态、Ban 和删除 API |

### 不适合

| 场景 | 原因 |
| --- | --- |
| 低延迟、实时请求处理 | Job Watcher 30 秒一轮，Task 抢占还要拿全局锁，端到端延迟是秒到分钟级 |
| 海量极小任务（每秒成千上万） | 全局 `taskAdoptionLock` 串行化抢占，且有 `maxWorkerCountOverall` 上限 |
| 要求 exactly-once 且任务无法做幂等 | 框架只保证至少一次，节点宕机后会重跑 |
| 需要严格顺序或强一致的分布式事务 | 框架不提供，Task 之间只有 Step 级别的依赖 |
| 带人工审批、复杂分支的业务工作流 | 这是 DAG 批执行框架，不是 BPM 引擎 |

### 我们的使用（待补）

- 财务报表 Job 的 Step 划分、分片数、触发方式（定时或手动）：待补。
- 幂等的具体做法（覆盖写、upsert、文件名带 trigger）：待补。
- 使用 `CAN_FAIL`、Routine、Node Duty、Ban 的实际情况：待补。
- 线上遇到的故障或重复执行事件：待补。

### 面试中被问到"为什么不用 X"

| 对比对象 | 可以这样答（先确认符合你们当时的实际情况） |
| --- | --- |
| Quartz / 单机定时任务 | 单机没有故障转移，也不能把一个任务拆成分片并行 |
| 带 leader 的调度框架（XXL-Job、Elastic-Job 等） | leader 要管理所有 Worker 的存活，设计更复杂；这里无 leader，Worker 自己抢，部署和扩容更简单 |
| Airflow / 工作流引擎 | 我们的任务嵌在业务应用里，以 library 形式使用，不想额外部署一套调度平台 |
| 消息队列消费 | 队列没有 Step 依赖、Job 状态、归档和重试管理，这些得自己实现 |

> 上表是通用对比，不是我们当时的决策记录，面试前需要你确认真实原因。
