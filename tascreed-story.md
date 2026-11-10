# TasCreed Story

架构图见 `tascreed-diagrams.md`。

## 故事主线

- 背景：财务报表每天要生成，数据量大，单机跑不完，也不能靠人盯着。
- 框架：TasCreed，轻量级分布式任务执行框架（Java / Spring Boot，etcd 做 bulletin）。
- 我们怎么用：定义 Job（Step DAG），Step 可分片并行；写 TaskExecutor；按 `jobName + trigger` 触发。
- 可靠性：无 leader，Worker 自己抢 Task，etcd lease + 心跳，节点挂了占用自动过期，别的节点重跑。
- 代价：至少一次执行，任务必须幂等。
- 我的贡献：待补（不要说成设计了框架，开源仓库唯一作者是 Lionel Liu）。

## 口述提示卡

1. 为什么需要：大、每天、要自愈
2. 模型：Job → Step（DAG）→ Task（sharding）
3. 无 leader：Worker 自己抢
4. etcd lease + 心跳 = 故障恢复
5. 至少一次 → 幂等
6. FAILED vs ERROR，CAN_FAIL
7. 我做了什么（待补）

## 30-second Overview

At eBay, my team used an internal distributed task framework called TasCreed to run our financial report jobs. It's a lightweight Java framework for reliable batch execution. You define a job as a DAG of steps, and each step can be split into shards that run in parallel across nodes. If a node crashes, another node picks up the task automatically, so we don't need to babysit the jobs. I worked on [待补: your part].

## Main Story in simple spoken English

**Why we needed it.** Our financial reports had to be generated every day, and they were too big for one machine. We needed to split the work, run it in parallel, and make sure it finished even if a node died.

**How it works.** A job is defined as a DAG of steps, for example "generate files by shard" and then "merge and upload". A step can be sharded into many tasks. There is no leader node. Every node runs the same code, and each worker picks up tasks by itself. The state of jobs and tasks lives in etcd.

**How failover works.** When a worker picks a task, it takes a lease in etcd and sends heartbeats to keep it alive. If the node crashes, the heartbeats stop, the lease expires, and etcd removes the claim automatically. Another node sees the task is free and runs it again. So there is no manual cleanup.

**The trade-off.** The framework guarantees at-least-once execution, not exactly-once. So our tasks must be idempotent. [待补: how we made them idempotent.]

**Failure handling.** A task result is either FAILED, an expected failure that can be retried, or ERROR, an unexpected exception. A step can be marked as allowed to fail, so it does not block later steps. Only successful jobs are archived. Failed ones stay visible, so we can retry them.

**My part.** [待补: your real contribution, e.g. defining report jobs and steps, writing task executors, fixing issues, operating it in production.]

## Technical Deep Dive

### Components
- **Job Server**: creates jobs from requests, rejects duplicates (same `jobName + trigger` in bulletin or archive), advances jobs, archives done jobs.
- **Task Worker**: a watcher thread picks a task, a worker thread executes it, a companion heartbeat thread keeps the lease alive.
- **Routine**: a long-running cluster-wide executor. The job watcher is a routine (`job-watcher`, scale 3, every 30 s).
- **Bulletin**: stores alive jobs, tasks, and adoptions (claims). Default is etcd.
- **Archive**: stores finished jobs. ES or etcd.

### Task pick and execution
1. The watcher takes a global `taskAdoptionLock` in etcd, so picking is serialized.
2. It reads all adoptions and todo tasks.
3. If adopted count reaches `maxWorkerCountOverall`, it stops.
4. It filters candidates by ban rules, affinity rules, and per-job share.
5. It picks the task with the highest weight (job priority).
6. `occupy`: grant a lease with TTL, then put the adoption key bound to the lease.
7. It releases the lock, runs the task, and heartbeats every TTL/3 with `keepAliveOnce`.
8. On finish, the worker updates the task state and revokes the lease.

### Failover
- Crash or hang: heartbeat stops, lease expires, etcd deletes the adoption key.
- The task is still todo and not claimed, so another node picks it up and reruns it.
- Heartbeat errors: retryable ones use backoff. A non-retryable error or `leaseId < 0` marks the occupation as not alive.

### States
- Task: `UNDONE`, `SUCCESS`, `FAILED`, `ERROR`. The user's TaskExecutor sets the state, the framework does not set it automatically.
- Step: `DORMANT`, `START`, `READY`, `SUCCESS`, `FAILED`, `ACCEPTABLE_FAILED`, `ERROR`, `IGNORED`, `SKIP_BY_FAILED`, `SKIP_BY_ERROR`.
- `FAILED` is retryable, `ERROR` is not. `CAN_FAIL` turns a failed step into `ACCEPTABLE_FAILED` with a success result.
- Job: `UNDONE`, `SUCCESS`, `FAILED`, `ERROR`, `STUCK` (circular dependency).
- Only `SUCCESS` jobs are archived. The others stay in bulletin and can be retried manually.

### Operations
- **Node Duty**: disable duties (`SERVER`, `EXECUTOR`, and finer ones) by TasCreed version, app version, or host name. Used for rollouts and multi-data-center setups.
- **Ban**: pause job submit, task creation, or task pick at different levels.
- **Schedule**: trigger job defines at a planned time.
- **Progression**: view job progress.

## Follow-up Questions

答案凡带 [待补] 的，面试前需要用你的真实经验替换。

### Design

**Q: Why a leaderless design?**
A leader has to track every worker's health to assign tasks, and that makes the design and implementation more complex. With no leader, every node has the same function and works on its own, so it's simple to deploy and scale. The cost is that we can't do smart assignment, such as by resource capacity.

**Q: Why etcd?**
It has native leases. A claim is a key bound to a lease, so if the worker dies, the claim expires on its own. With a normal database, we'd have to build the timeout and cleanup ourselves.

**Q: Why not Quartz, XXL-Job, Airflow, or a message queue?** [待补: real reason]
Quartz on its own has no sharding or cross-node failover. Leader-based schedulers need leader and worker health management. Airflow means running a separate platform, while TasCreed is a library inside our application. A queue has no step dependencies, job state, archive, or retry management.

**Q: How does a job with dependent steps run?**
The job watcher reads alive jobs and done tasks. When all dependencies of a step are done, it updates the step state and creates tasks for that step. Sharded steps create N tasks at once.

### Reliability

**Q: What does at-least-once mean for your tasks?**
After a crash, a task can run again, maybe partially done before. So the task must give the same result when repeated. [待补: e.g. overwrite output for a given date, upsert by key.]

**Q: What if a worker hangs but keeps sending heartbeats?**
Heartbeats come from a separate thread, so a stuck task could keep its lease. That is a real risk. Idempotency and task-level timeouts and monitoring are the safety net. [待补: what we did in practice.]

**Q: What if etcd is down or slow?**
Heartbeats retry with backoff. If etcd is unavailable longer than the TTL, leases expire, and tasks may be picked up again, so idempotency matters again. Job creation and updates fail with retryable errors.

**Q: Can two workers run the same task at the same time?**
The pick is protected by a global lock and the adoption key, so normally no. But after a lease expires while the old worker is slow or partitioned, the old worker may still be running when a new one starts. That's why tasks must be idempotent.

**Q: What is the difference between FAILED and ERROR?**
FAILED is an expected failure and is usually retryable. It only fails the current and dependent steps, and other steps keep going. ERROR is an unexpected exception. It marks the job as ERROR, and the remaining steps will not run, except tasks that are already created.

**Q: How did you recover from a failed job?**
Failed jobs stay in bulletin, not archive. We could retry them manually or delete and resubmit. [待补: actual process.]

### Scale and performance

**Q: Is the global lock a bottleneck?**
It serializes task picking. For tasks that run for minutes, picking is rare, so it's fine. It would not suit huge numbers of tiny tasks.

**Q: How do you control parallelism?**
`maxWorkerCountOverall` limits adopted tasks, sharding sets the number of tasks per step, and job priority decides order. Ban and affinity rules add more control.

**Q: How did you scale it?**
Add more nodes. Each node runs the same code, and they compete for tasks. [待补: node count, shard count.]

**Q: How long does it take from task creation to pick-up?**
The job watcher runs every 30 s, and task watchers poll periodically. So the latency is seconds to a minute, which is fine for batch jobs. [Confirm the actual settings.]

### Operations

**Q: How did you deploy new versions safely?**
Node duty rules can disable duties on nodes that are below a minimum TasCreed or app version, or by host name. That's useful during a rollout when several versions run together.

**Q: How did you monitor it?**
There are metrics and a progression view of jobs. [待补: dashboards and alerts we used.]

**Q: What did you do when a job got stuck?**
Check the job and step states, find the failing task, look at its logs and adoption. Then retry or fix and resubmit. [待补: a real case.]

### Behavioral

**Q: What was your contribution?** [待补]
Be specific: what you built, what you fixed, what you owned. Use "I" for your own work and "we" for the team.

**Q: What was the hardest problem?** [待补]
Possible angles: duplicate execution after failover, idempotency of report output, stuck jobs, upgrade compatibility.

**Q: What would you improve?**
Possible answers (these are design observations, not facts about our system): reduce the global pick lock for higher throughput, add stronger protection against hung workers, and fence tasks so an old worker cannot write after its lease expires.

**Q: Is this open source? Did you write it?**
"The framework has an open-source version from eBay. I worked on [待补] in our internal version and on using it in the report subsystem. The open-source project was led by its author, Lionel Liu."

## Useful Expressions

- reliable distributed task execution
- leaderless design
- lease and heartbeat
- at-least-once, so tasks must be idempotent
- failover without manual intervention
- a DAG of steps, sharded into tasks
- the claim expires on its own
- an expected failure vs. an unexpected exception
- a safety net

## 下一轮需要补齐的事实

1. 我的具体贡献（哪些模块或功能，开发、使用，还是运维）。
2. 财务报表 Job 的 Step 划分、分片数、触发方式、运行时长。
3. 幂等的具体做法。
4. 是否用了 CAN_FAIL、Routine、Node Duty、Ban、Schedule。
5. 内部版本与开源版本的差异（etcd 使用方式等）。
6. 当时为什么选 TasCreed 而不是其他方案。
7. 线上遇到过的故障、重复执行、卡死案例。
8. 监控和告警方式。
9. 节点数、规模、etcd 集群情况。
10. 这个框架是否就是简历中"Java multithreading framework with failover and etcd service registration"。
