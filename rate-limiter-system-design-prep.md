# Rate Limiter – System Design Cheat Sheet

## One-liner
> "A rate limiter protects the platform and keeps merchants fair. I use a token bucket per merchant, with shared state in Redis updated atomically, and it fails open if Redis is down."

## 1. Clarify
- Limit by what? Merchant / API key / IP / endpoint.
- Hard limit (429) or soft (queue)?
- Scale: requests per second, number of merchants?
- Latency budget? (It is on the critical path: aim for ~1-2 ms.)
- Limiter down: fail open or fail closed?
- Single or multi region? Does the limit need to be exact?

## 2. Algorithm: Token Bucket
- Bucket has a capacity (max burst) and a refill rate (average rate).
- Each request takes 1 token. No token → 429.
- State per key is tiny: `tokens` + `lastRefillTime`.
- Refill lazily on each request: `tokens = min(cap, tokens + elapsed * rate)`.

| Algorithm | Problem |
|---|---|
| Fixed window | Boundary burst: up to 2x the limit |
| Sliding window log | Exact but memory heavy |
| Sliding window counter | Good approximation, a bit more complex |
| Leaky bucket | Smooth output, no burst |

Say: "Payments are bursty, so token bucket fits: it allows short bursts but caps the average."

## 3. Where it runs
```
Client -> API Gateway (rate limit check) -> Payment Service
                |
              Redis (token buckets, key = merchant_id:api)
                |
        Config store (limits per merchant / plan)
```
- Check at the gateway, before any expensive work.
- Limits come from the contract / plan, cached in memory and refreshed.
- Response: `429` + `Retry-After`, plus `X-RateLimit-Limit/Remaining/Reset` headers.

## 4. Shared state across nodes
- Many stateless gateway nodes → one central Redis.
- Use a **Lua script** so read + refill + decrement is **one atomic step**. No race between nodes.
- Not `limit / N` per node: uneven traffic would block valid requests.
- Not sticky routing: bad for failover and load balancing.

## 5. Failure handling
- Redis slow or down → **fail open** with a short timeout (e.g. 5 ms).
- Fall back to a coarse local in-memory limit, and alert.
- Reason: the limiter is a protection, not the business. Do not stop all payments because of it.
- Exception: fail closed for abuse-sensitive endpoints if needed.

## 6. Hot keys
- A huge merchant can overload one Redis shard.
- Option A: split into N sub-buckets (`merchant:1..N`), each with limit/N.
- Option B: each gateway node takes a batch of tokens locally and syncs with Redis periodically. Fewer Redis calls, slightly less exact.
- Trade-off: exactness vs. load.

## 7. Multi-region
- Simple: each region has its own limit (global limit / regions).
- Exact global limit needs cross-region sync: slow, usually not worth it.
- Accept approximation. Say it clearly.

## 8. Layers of protection
1. Edge / WAF: IP-level and DDoS.
2. Gateway: per merchant, per API.
3. Service: concurrency limit and load shedding for downstream (bank) protection.

## One-minute answer
"First I clarify what we limit and how exact it must be. I choose token bucket per merchant, because it allows bursts and uses little memory. The gateway checks Redis with an atomic Lua script, so many nodes share one counter without races. Limits come from the merchant contract. If Redis fails, we fail open with a local fallback and alert. For hot merchants, I split the key into sub-buckets or lease tokens locally. Over the limit, we return 429 with Retry-After."

## Follow-ups
| Question | Short answer |
|---|---|
| Why not a DB? | Too slow for every request; Redis is in-memory and atomic. |
| Why Lua? | Atomic read-modify-write, no race. |
| Exactly accurate? | No. Trade a small error for speed and availability. |
| Different limits per merchant? | Config store keyed by merchant plan, cached at the gateway. |
| Client retries after 429? | `Retry-After` plus client backoff with jitter. |
| Protect the bank? | Separate downstream concurrency limit + circuit breaker. |
| Metrics? | Allowed / rejected per merchant, Redis latency, fail-open count. |

## Remember
- Token bucket + Redis + Lua.
- Fail open, not closed.
- Hot key: split or lease tokens.
- Approximate is fine; say the trade-off.
