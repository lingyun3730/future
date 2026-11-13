# Distributed Task Scheduler – System Design Cheat Sheet

## One-liner
> "Each trigger is a row in a DB. Scheduler instances claim due rows atomically with a lease. Workers run them. If an owner dies, the lease expires and another instance takes over. Delivery is at-least-once, so jobs must be idempotent."

## 1. Clarify
- Job types: cron, one-time, delayed? Short or long running?
- Scale: number of jobs, triggers per second?
- Semantics: at-least-once or exactly-once? Are jobs idempotent?
- Timing: is a few seconds of delay OK?
- Does the scheduler run the job, or only trigger a worker / service?
- Retry, timeout, priority?

## 2. Data model
```
job            (job_id, name, cron_expr, payload, owner_service, enabled)
job_trigger    (job_id, fire_time, status, owner, lease_expires_at,
                attempt, next_retry_at)
               UNIQUE (job_id, fire_time)
               INDEX  (status, next_fire_time / lease_expires_at)
```
- One row per planned run = the trigger is also the task.
- `status`: PENDING -> RUNNING -> SUCCESS / FAILED (or back to PENDING for retry).

## 3. Flow
```
Planner:   create next trigger rows (job_id + fire_time, unique)
              |
Scheduler: claim due rows  (UPDATE ... SET owner=me, lease=now+30s
              |             WHERE status=PENDING AND fire_time<=now)
              |             or SELECT ... FOR UPDATE SKIP LOCKED
              v
Worker:    run job, heartbeat renews lease
              |
           mark SUCCESS / FAILED (retry with backoff + jitter)
```

## 4. Key points
1. **No duplicate trigger:** unique `(job_id, fire_time)` + atomic claim. The DB decides one winner, no leader needed.
2. **Failover:** lease + heartbeat. Lease expired -> another instance reclaims the row and re-runs it.
3. **At-least-once:** a crash can cause a re-run. So jobs must be idempotent. Dedupe key = `job_id + fire_time`.
4. **Exactly-once?** Not truly. Say: "at-least-once delivery + idempotent job = effectively once."
5. **Retry:** exponential backoff with jitter, max attempts, then FAILED + alert.
6. **Misfire** (system was down at fire time): policy per job: run once now / skip / run all missed.
7. **Long jobs:** heartbeat extends the lease; set a max runtime; support cancel (kill) from an API.

## 5. Scale
- Shard jobs by hash(job_id). Each scheduler instance owns some shards and scans only those.
- Index on `(status, fire_time)` so the "due" scan is cheap.
- Separate scheduler (decides when) from workers (do the work). Scale workers independently.
- Use a message queue between them if the volume is high.
- Add per-tenant concurrency limits so one tenant cannot starve others.

## 6. Why not a leader?
- A single leader is simple but a bottleneck and a failover pause.
- DB-claim lets every instance work in parallel and the DB resolves conflicts.
- Leader election (ZooKeeper / etcd) is fine for small scale. Say the trade-off.

## 7. Operate
- Metrics: scheduling delay (fire_time vs actual start), backlog of due triggers, failure rate, retries, expired leases.
- Alert on delay > threshold and on stuck RUNNING triggers.
- Admin: pause / resume / re-run / kill a job; audit history.

## One-minute answer
"I clarify job types, scale and delivery semantics first. Every planned run is a row with a unique job id plus fire time, so a trigger is never created twice. Scheduler instances claim due rows atomically with a lease, using a conditional update or SKIP LOCKED, so only one wins. Workers renew the lease with heartbeats; if one dies, the lease expires and another instance re-runs the task. That gives at-least-once, so jobs are idempotent with job id plus fire time as the dedupe key. Retries use exponential backoff with jitter. To scale, I shard by job id, keep scheduling and execution separate, and monitor scheduling delay and backlog."

## Follow-ups
| Question | Short answer |
|---|---|
| Exactly-once? | No. At-least-once + idempotent jobs. |
| Two instances claim the same row? | Atomic conditional update: only one gets 1 row affected. |
| Clock skew between machines? | Use DB time for fire_time and lease checks. |
| Lease too short / too long? | Short = false takeovers; long = slow failover. Use ~3x heartbeat interval. |
| Job runs longer than lease? | Heartbeat keeps extending it. |
| Thundering herd at midnight? | Spread with jitter, per-tenant limits, queue between scheduler and workers. |
| Dependencies between jobs (DAG)? | Add a dependency table; trigger next when all upstream are SUCCESS. |
| Priority? | Priority column + separate queues; avoid starvation with aging. |

## My experience: TasCreed (use as a real-world example)
TasCreed is an internal distributed task framework at eBay. We used it to run daily financial reports. I was a heavy user, not the original designer: say this honestly.

How it maps to this design:
| This design | TasCreed |
|---|---|
| Trigger row, unique key | `jobName + trigger`, duplicates rejected |
| Claim by DB / atomic update | Worker takes a claim (adoption) in etcd, no leader |
| Lease + heartbeat | etcd lease with TTL, heartbeat every TTL/3 |
| Failover | Node dies -> lease expires -> another node re-runs |
| Shards | A job is a DAG of steps; a step splits into parallel tasks |
| At-least-once | Yes, so our tasks had to be idempotent |

What I added (STAR in 3 lines):
- Problem: some tasks retried forever and I could not stop them in time.
- Action: added exponential backoff to the base framework, plus an API to kill a retrying thread. Later the team built a portal (with permissions) to monitor and kill them centrally.
- Lesson: at-least-once + retry can loop forever. Backoff limits the damage; a kill switch gives humans the last word.

Say in the interview:
> "In my previous team we used a framework like this for daily financial reports. It had no leader: workers claimed tasks with etcd leases. I improved retries with exponential backoff and a kill API. That's why I design for bounded retries and an operator override."

Trade-off to mention: no leader = simple and scalable, but no smart assignment by capacity. A hung task can keep heartbeating, so add task timeouts.

Full details: `../project-coding-examples/tascreed-story.md`.

## Remember
- Trigger row + unique key + atomic claim.
- Lease + heartbeat = failover.
- At-least-once + idempotent = effectively once.
- Shard by job id; split scheduler and workers.
