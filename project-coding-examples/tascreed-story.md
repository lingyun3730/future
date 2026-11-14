# TasCreed Story

架构图见 `tascreed-diagrams.md`。

## 故事主线

- 背景：财务报表每天要生成，数据量大，单机跑不完，也不能靠人盯着。
- 框架：TasCreed，轻量级分布式任务执行框架（Java / Spring Boot，etcd 做 bulletin）。
- 我们怎么用：定义 Job（Step DAG），Step 可分片并行；写 TaskExecutor；按 `jobName + trigger` 触发。
- 可靠性：无 leader，Worker 自己抢 Task，etcd lease + 心跳，节点挂了占用自动过期，别的节点重跑。
- 代价：至少一次执行，任务必须幂等。
- 我的角色：财务报表场景的重度使用者；调试时发现框架问题就主动修复（不要说成设计了框架，开源仓库唯一作者是 Lionel Liu）。
- 我的改进 1：调试时发现有线程无限 retry 又停不下来 → 在基础框架加 retry 策略（exponential backoff），并开放 API 可以 kill 掉一直 retry 的线程。
- 我的改进 2：用的人变多后，团队做了 UI portal（etcd 的可视化界面：当前有哪些 job 在跑、有哪些 task adoptions），集中监控和管理正在运行的线程，发现无效 retry 的线程就在 portal 上 kill；UI 有权限控制。（我在 UI 里的具体角色：待补）

## 口述提示卡

1. 为什么需要：大、每天、要自愈
2. 模型：Job → Step（DAG）→ Task（sharding）
3. 无 leader：Worker 自己抢
4. etcd lease + 心跳 = 故障恢复
5. 至少一次 → 幂等
6. FAILED vs ERROR，CAN_FAIL
7. 我做了什么：发现无限 retry → 加 exponential backoff + kill API → 后来团队做 portal 集中监控和 kill（带权限）

## 30-second Overview

At eBay, my team used an internal distributed task framework called TasCreed to run our financial report jobs. It's a lightweight Java framework for reliable batch execution. You define a job as a DAG of steps, and each step can be split into shards that run in parallel across nodes. If a node crashes, another node picks up the task automatically, so we don't need to babysit the jobs. I worked on [待补: your part].

## Main Story in simple spoken English

**Why we needed it.** Our financial reports had to be generated every day, and they were too big for one machine. We needed to split the work, run it in parallel, and make sure it finished even if a node died.

**How it works.** A job is defined as a DAG of steps, for example "generate files by shard" and then "merge and upload". A step can be sharded into many tasks. There is no leader node. Every node runs the same code, and each worker picks up tasks by itself. The state of jobs and tasks lives in etcd.

**How failover works.** When a worker picks a task, it takes a lease in etcd and sends heartbeats to keep it alive. If the node crashes, the heartbeats stop, the lease expires, and etcd removes the claim automatically. Another node sees the task is free and runs it again. So there is no manual cleanup.

**The trade-off.** The framework guarantees at-least-once execution, not exactly-once. So our tasks must be idempotent. Idempotency depends on the task. For example, each shard of a report writes a file with a fixed name, so a re-run overwrites the old file and never creates a duplicate.

**Failure handling.** A task result is either FAILED, an expected failure that can be retried, or ERROR, an unexpected exception. A step can be marked as allowed to fail, so it does not block later steps. Only successful jobs are archived. Failed ones stay visible, so we can retry them.

**My part.** I was a heavy user of the framework, mainly for generating our financial reports. When I found something to improve while debugging, I fixed it in the framework myself. For example, I noticed some threads were retrying endlessly and I had no way to stop them in time. So I added retry strategies to the base framework, such as exponential backoff, and I exposed an API to kill threads that keep retrying. Later, more teams started using the framework, so we built a UI portal to monitor and manage the running threads in one place. The portal shows the data in etcd, such as which jobs are running and which tasks are adopted. If a task is stuck, authorized people can kill its adoption from the portal. The portal has permission control, so only authorized people can do that. [待补: your exact role in the portal.]

### My improvement story (STAR)

- **Situation:** While debugging report jobs, I saw threads retrying endlessly, for example when a dependency was down or a task could never succeed. They kept using resources and I couldn't stop them in time.
- **Task:** Make retries smarter and give operators a way to stop a bad retry loop without redeploying.
- **Action:** I added more retry strategies to the base framework, including exponential backoff, so retries slow down instead of hammering the dependency. I also exposed an API to kill a thread that keeps retrying. It deletes the task's adoption in etcd, so on the next heartbeat the worker can't find it and the task is released.
- **Result:** We could stop runaway retries quickly. As adoption grew, the team built a portal with permission control to monitor and kill threads centrally. [待补: numbers, e.g. incidents avoided, number of teams.]

### Why this matters (tie it to the architecture)

"Because the framework is at-least-once and retry-based, a task that can never succeed can loop forever. Backoff limits the damage, and the kill switch gives humans the last word."

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
Because there is no leader, many workers compete for the same task at the same time, and we need exactly one of them to win. That needs strongly consistent coordination: every node must see the same answer, and a write is visible to all readers right after it succeeds. etcd is built on Raft, so it gives us that. Its atomic "create this key only if it doesn't exist yet" operation lets exactly one worker win, with no duplicates.

The second reason is failure handling. etcd has leases, which work like a timer. A worker grabs a task by writing a key, "I'm taking task X", and attaches a lease to it, for example 10 seconds. While the worker is alive, it keeps renewing the lease. If the worker crashes, it stops renewing, the lease expires, and etcd deletes the key. The task is then free again and another worker can take it. With a normal database, we would have to build this timeout and cleanup ourselves.

In plain words: etcd makes sure only one worker takes each task, and if that worker dies, the task is released automatically.

**Q: Why not Quartz, XXL-Job, Airflow, or a message queue?** [待补: real reason]
Quartz on its own has no sharding or cross-node failover. Leader-based schedulers need leader and worker health management. Airflow means running a separate platform, while TasCreed is a library inside our application. A queue has no step dependencies, job state, archive, or retry management.

**Q: How does a job with dependent steps run?**
The job watcher reads alive jobs and done tasks. When all dependencies of a step are done, it updates the step state and creates tasks for that step. Sharded steps create N tasks at once.

### Reliability

**Q: What does at-least-once mean for your tasks?**
After a crash, a task can run again, maybe partially done before. So the task must give the same result when repeated. It depends on the business case. For our report shards, the output file name is fixed per shard, so a re-run overwrites the old file instead of creating a new one.

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

**Q: What was your contribution?**
"I was a heavy user, mainly for financial reports. When I found a problem, I fixed it in the framework. The main one was endless retries: I added retry strategies like exponential backoff and an API to kill a retrying thread. Later we built a portal, with permission control, to monitor and kill threads centrally."

**Q: Why add exponential backoff?**
With a fixed retry interval, a failing dependency gets hit again and again at the same rate. With exponential backoff, the wait grows after each failure, so it reduces load and gives the dependency time to recover. [待补: the max delay and max retry count you chose, and whether you added jitter.]

**Q: Why not just limit the retry count?**
A count limit helps, but some failures are temporary and long, like an etcd or downstream outage, so we still want to retry for a while. Backoff handles that. The kill API covers cases we didn't predict. [Check this matches what you did.]

**Q: How does the kill API work?**
"It deletes the task's adoption key in etcd. On its next heartbeat, the worker reads the adoption key, finds it missing or different, and the heartbeat fails with a non-retryable error. The occupation is marked as no longer alive, so the task is released. This reuses the same lease-and-heartbeat mechanism that handles node crashes, so we didn't need a new kill channel." [待补: how the worker thread itself stops after losing the occupation, and how long it takes, up to one heartbeat period, which is about TTL/3.]
Be ready for: killing is not cancelling the job. The task is still todo, so another node may pick it up again. To really stop it, ban or delete the job. Also, the kill takes effect at the next heartbeat, not instantly.

**Q: What happens to the task after a kill?**
"The adoption is gone, so the task is released and is still todo. Another worker can pick it up and run it again, which is safe because tasks are idempotent. If we want it to stop for good, we ban the job." [待补: what you actually did in practice.]

**Q: How did you decide which threads to kill?**
We watched the portal for threads stuck in repeated retries with the same error and no progress. [待补: what signals you used, such as retry count, running time, or error message.]

**Q: Why build a portal?**
As more people used the framework, calling APIs by hand did not scale. A central UI shows all running threads, and operators can act quickly.

**Q: How is the portal secured?**
It has permission control, so only authorized users can view or kill threads. Killing is a risky operation. [待补: the model, e.g. roles, per-team access, audit log.]

**Q: Did you contribute your changes back to the open-source version?**
[待补.] The open-source repo has retry strategies (equal, linear backoff, backoff) and API to delete a task adoption. Check with your own history whether these are yours before claiming them.

**Q: Tell me about a time you improved a tool you were using.**
Use the retry story above. Lead with the pain (endless retries), then what you built, then how it grew into a team portal.

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

1. 我的贡献细节（已知：加了 exponential backoff 等 retry 策略和 kill API；团队做了带权限的 portal）。还缺：
   - kill 之后 Worker 线程本身怎么停下来、多久生效（已知：删 etcd 上的 adoption，心跳找不到后释放 Task）
   - backoff 的参数（初始间隔、上限、最大次数、是否有 jitter）
   - portal 里我具体做了什么（前端、后端、需求、权限设计？）
   - 哪些改动合进了开源版本（`ACKNOWLEDGEMENTS.md` 里有 Lintao Dang，但 Contributors 里没有）
   - 当时触发这件事的具体事件，以及改进后的效果和使用人数
2. 财务报表 Job 的 Step 划分、分片数、触发方式、运行时长。
3. 幂等的具体做法。
4. 是否用了 CAN_FAIL、Routine、Node Duty、Ban、Schedule。
5. 内部版本与开源版本的差异（etcd 使用方式等）。
6. 当时为什么选 TasCreed 而不是其他方案。
7. 线上遇到过的故障、重复执行、卡死案例。
8. 监控和告警方式。
9. 节点数、规模、etcd 集群情况。
10. 这个框架是否就是简历中"Java multithreading framework with failover and etcd service registration"。

## Coding principles with examples（第一轮用）

| 原则 | 我的例子 |
| --- | --- |
| 为失败设计：有界重试 | 发现线程无限重试，给基础框架加了指数退避，重试越来越慢，避免打爆依赖 |
| 永远留人工开关 | 退避只限制损害，所以加了 kill API；后来 portal 让有权限的同事 kill 卡住的 task adoption |
| 在根上修，不绕开 | 调试时发现问题，直接改基础框架，所有团队受益，而不是在每个报表里各自处理 |
| 重复执行要安全（幂等） | 框架是 at-least-once，每个 shard 的输出文件名固定，重跑覆盖旧文件，不产生重复 |

> One principle I follow is designing for failure. In our task framework, some tasks retried endlessly. I added exponential backoff so retries slow down instead of hammering the dependency. But backoff only limits the damage, so I also added a kill API, so people always have a manual override. I fixed it in the framework itself, so every team benefited. And since the framework runs tasks at least once, each report shard writes to a fixed file name, so a re-run just overwrites the old file.
