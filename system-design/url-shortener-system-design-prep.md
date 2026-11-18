# URL Shortener – System Design Cheat Sheet

## One-liner
> "A URL shortener maps a short code to a long URL. It's read-heavy, so I generate unique codes with an ID generator plus base62, store the mapping in a key-value store, and serve redirects from a cache."
>
> 短链接服务：短码 → 长 URL 的映射。读多写少，所以用唯一 ID + base62 生成短码，映射存 KV 库，读走缓存。

## 0. 一页口诀（复述顺序，先看这个）
**问规模 → 估算 → API → 生成短码 → 存储 → 缓存 → 状态码 → 统计 → 过期与防滥用**

| 步骤 | 一句话 | 关键数字或词 |
| --- | --- | --- |
| 1 问规模 | 每月新增多少、读写比、保存几年、URL 多长、要不要统计和过期 | 例：1 亿/月，读写 100:1，5 年 |
| 2 估算 | 写 ≈ 38/s，读 ≈ 3800/s，约 60 亿条，约 3 TB；**读多写少，重点是缓存** | 62^6 ≈ 568 亿，62^7 ≈ 3.5 万亿 → 6 到 7 位 |
| 3 API | 创建：长 URL → 短 URL；访问：短码 → 重定向 | |
| 4 生成短码 | **ID 来源 → 打乱 → base62**。哈希截取会碰撞，所以不用；单个自增计数器是单点且可猜测 | 见第 4 节 |
| 5 存储 | **短码做主键**，KV 或分片 SQL，**按短码哈希分片** | 点查，无 join，无事务 |
| 6 缓存 | Bloom filter → Redis（LRU，cache-aside）→ DB；防**缓存穿透** | 新短码必须加进 Bloom filter |
| 7 状态码 | 要统计、要过期，选 **302**；301 会被浏览器缓存，统计不到 | |
| 8 统计 | 点击事件**异步**发 Kafka，消费者批量聚合，不能拖慢跳转 | 可用性优先于精确 |
| 9 过期与防滥用 | 读时检查过期 + 后台批量清理；认证、限流、URL 校验、黑名单 | |

**短码生成的推导链（重点，最容易忘）：**
1. 单个数据库自增 → **单点故障**，而且 id 连续 → **可被猜测、可被枚举**。
2. 解决单点：**号段分配**（每台服务器一次领 1000 个 id）或**自定义 Snowflake**。
3. 标准 Snowflake 是 64 位 → base62 要 **11 位**，太长。要缩短就压缩各部分位数（时间戳用秒级、机器位和序列位变少），目标 36 到 42 位。
4. 解决可猜测：对 id 做**双射变换**（如乘一个大奇数再模 2^n），结果打乱但**一一对应，不会重复**；哈希截取是多对一，会碰撞。
5. 最后 base62 编码。

## 0.1 English walkthrough (speak it in this order)

**1. Clarify.** "How many new URLs per month, what's the read-to-write ratio, how long do we keep them, how long is a URL, and do we need analytics and expiration?" Assume 100M per month, 100:1 reads to writes, 5 years.

**2. Estimate.** "That's about 38 writes per second and about 3,800 reads per second, so it's read-heavy and the cache matters most. Over 5 years that's about 6 billion records, roughly 3 TB, which is small. With base62, 62^6 is about 56.8 billion, so 6 characters is enough, and I'd pick 7 for headroom."

**3. API.** "One endpoint to create: long URL in, short URL out. One endpoint to redirect: short code in, redirect to the long URL."

**4. Generate the code.** "I don't truncate a hash, because that can collide. I don't use a single database counter either, because it's a single point of failure and the IDs are sequential, so people can guess and enumerate links. Instead, each app server takes a range of IDs, say 1,000 at a time, from a central store, so IDs are unique and short. A Snowflake-style ID also works, but a 64-bit ID needs 11 base62 characters, so I'd need a custom, shorter layout. Then I scramble the ID with a one-to-one mapping, for example multiplying by a large odd number modulo 2^n, so codes look random but never collide. Finally I encode it in base62."

**5. Storage.** "The only lookup is by code, so the code is the primary key. There are no joins and no transactions, so a key-value store fits well, and a sharded relational database also works. I shard by hash of the code, using consistent hashing or many logical shards so I can add machines without moving most of the data."

**6. Caching.** "Redirects go through a Bloom filter, then Redis with LRU, then the database, using cache-aside. The Bloom filter protects against cache penetration: requests for codes that don't exist are rejected right away instead of hitting the database. When a new link is created, I add its code to the Bloom filter, so it never gives a false negative."

**7. Status code.** "I use 302. A 301 is cached by the browser, so later clicks never reach us, which breaks click counting and makes it hard to change or expire a link."

**8. Analytics.** "I publish a click event to Kafka asynchronously and return the redirect immediately. A consumer aggregates the events in batches. Redirect availability matters more than exact counts."

**9. Expiration and abuse.** "I store an expiry time, check it on read, and purge expired rows in a background job. For abuse, I require authentication, rate limit creation per user or IP, validate URLs, and check them against a blocklist."

## 1. Clarify（先问清楚）
- Features: create short link, redirect. Custom alias? Expiration? Analytics (click count)? Delete or edit?
- Scale: new URLs per day, read:write ratio? (Typical: 100:1 reads vs writes.)
- Short code length? Can the same long URL map to the same code, or a new code each time?
- Predictable codes OK, or must they be hard to guess?
- Availability vs consistency: a redirect must almost never fail.

## 2. Estimation（估算，用来选方案）
Example: 100M new URLs per month, 100:1 read ratio.
- Writes: 100M / (30 × 86400) ≈ 40 per second.
- Reads: ≈ 4,000 per second on average, peaks several times higher.
- Storage: ~500 bytes per record × 100M × 12 months × 5 years ≈ 3 TB. Small. A sharded store is easy.
- Code length: base62 has 62 characters (0-9, a-z, A-Z). 62^6 ≈ 56.8 billion, which already covers 5 years (about 6 billion URLs) with roughly 9x headroom. 62^7 ≈ 3.5 trillion. **6 characters is enough here, and 7 is a common choice for extra headroom.**

Say: "The system is read-heavy and the data is small, so the cache and the redirect path matter most."

## 3. API
```
POST /api/v1/urls        { "longUrl": "...", "alias": "optional", "expireAt": "optional" }
                         -> { "shortUrl": "https://sho.rt/aZ3k9Q1" }
GET  /{code}             -> 301 or 302 redirect to the long URL
```

## 4. How to generate the short code（核心）

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Counter + base62** | Get a unique auto-increment ID, encode it in base62 | No collision, simple, short | IDs are predictable (code can be guessed and enumerated); one counter is a bottleneck |
| **B. Hash + truncate** | MD5 or SHA of the long URL, take the first 7 chars | Same URL gives the same code | **Collisions** — need a check and retry; extra DB lookup |
| **C. Random code** | Generate a random 7-char string | Hard to guess | Collisions; needs a uniqueness check |
| **D. Pre-generated key service** | A key-generation service creates unique codes in advance and hands out batches | No collision at request time, fast | Another component to run |

**My choice:** Unique ID → scramble → base62.

**Problems with a single DB auto-increment counter:**
- **Single point of failure**, and hard to scale across regions.
- IDs are sequential, so codes are **guessable**: anyone can enumerate other people's links.

**Step 1: get a unique ID without a single counter. Two options:**
- **ID range allocation (simple, short IDs):** each app server takes a batch (for example 1,000 IDs) from a central store (ZooKeeper, Redis `INCRBY`, or a DB sequence) and uses them locally. The central store is hit rarely, and a server can keep working on its leftover range if the store is briefly down. 6 billion IDs need only about 33 bits, which is 6 base62 characters.
- **Snowflake-style ID generator:** `1 sign bit | 41-bit timestamp (ms) | 10-bit machine id | 12-bit sequence`. Unique as long as machine ids are unique and the clock doesn't go backwards. **Trade-off:** a 64-bit ID needs **11 base62 characters**, too long for a "short" link. To shorten it, make a custom layout: use second-level timestamps with a custom epoch (about 28 bits for 8 years), fewer machine bits, and fewer sequence bits, targeting 36 to 42 bits in total. Note the sequence is only a small part (12 bits), so shrinking it alone doesn't help much; the timestamp precision is the big saving.

**Step 2: make codes hard to guess.** Sequential or time-ordered IDs are still predictable, and a Snowflake timestamp is not a secret. Apply a **reversible one-to-one mapping (a bijection)** before base62, so the code looks random:
- Multiply by a large **odd** number modulo 2^n. An odd number is coprime with 2^n, so every ID maps to a different result. Small example: modulo 10 and multiply by 3 gives 0,3,6,9,2,5,8,1,4,7, all different.
- Other options: XOR with a secret (weak), shuffle the base62 alphabet (weak), a Feistel network or format-preserving encryption (stronger).
- **Why it never collides:** the mapping is a permutation of the ID range. Truncating a hash is many-to-one, so it can collide.

**Step 3:** encode in base62.

Say: "I get a unique ID from range allocation, scramble it with a one-to-one mapping so it can't be guessed, and encode it in base62. Because the mapping is a bijection, two IDs never give the same code."

**Base62 encoding（容易理解版）：**
```java
static final String CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

static String encode(long id) {
    StringBuilder sb = new StringBuilder();
    while (id > 0) {
        sb.append(CHARS.charAt((int) (id % 62)));
        id /= 62;
    }
    return sb.reverse().toString();
}
```

**Hash 方案的碰撞处理：** 如果用哈希，发现 code 已存在且对应不同的 URL，就换盐（加一个计数后缀）重新哈希。

## 5. Data model（数据模型）
Key-value is enough. One table (or key-value collection):

| Field | Note |
|---|---|
| `code` (primary key) | the short code |
| `long_url` | |
| `created_at`, `expire_at` | |
| `user_id` | optional |

- **Primary key = `code`.** The only lookup is "find the long URL by code", so no extra auto-increment id and no extra index is needed. Store the code, not the full short URL with the domain.
- Creation needs **no lookup**: the ID is already unique, so just insert. A lookup is needed only for custom aliases (insert "only if not exists", return 409 if taken) or to reuse the code of an existing long URL.
- Store choice: **a NoSQL key-value store (DynamoDB, Cassandra) or a sharded SQL DB.** Both work. The access pattern is pure primary-key lookups with no joins and no transactions, so a KV store scales most naturally. A sharded relational DB is also acceptable.
- **Sharding:** shard by hash of `code`. Because the scrambling makes codes evenly distributed, there are no hot shards. Plain `hash % N` forces most data to move when N changes, so use **consistent hashing** or **many logical shards** (for example 1,024) mapped to physical machines, so adding a machine only moves some shards.
- **Uniqueness:** the primary key guarantees it.

## 6. Architecture
```
Client → CDN / Load balancer → App servers (stateless)
                                   ├─ Cache (Redis)  ← redirect reads go here first
                                   └─ DB (sharded KV)
                                   └─ ID generator / key service (writes)
Click events → Kafka → analytics (async, off the redirect path)
```

**Redirect flow:**
1. `GET /aZ3k9Q1` → check the **Bloom filter**. "Definitely not present" → return 404 right away.
2. Otherwise look up the code in Redis.
3. Hit → return the redirect. Miss → read the DB, fill the cache, return the redirect.
4. Not found or expired → 404 (or 410). A false positive from the Bloom filter only costs one extra DB lookup.

**Create flow:** validate URL → get unique ID → scramble → base62 → write to DB → **add the code to the Bloom filter** (otherwise the new link would be rejected as "not present") → return the short URL.

## 7. Caching（读路径优化）
- 80/20 rule: a small share of links gets most of the traffic. Cache the hot ones with **LRU**, TTL as a safety.
- **Cache-aside** pattern: read the cache first, load from the DB on a miss.
- **Cache penetration**（缓存穿透）: requests for codes that don't exist never hit the cache, so every one reaches the DB (random guessing or a crawler can take the DB down). Fix: put a **Bloom filter** in front (a "not present" answer is always right, so reject immediately; add every new code on creation), and/or **cache the "not found" result with a short TTL**, and rate limit abusive IPs. See `bloom-filter-cheat-sheet.md`.
- Hot key: a viral link goes to one cache node. Fix: local in-process cache (Caffeine) in front of Redis, or replicate hot keys.

## 8. 301 vs 302（常被问）

| | 301 Moved Permanently | 302 Found (temporary) |
|---|---|---|
| Browser caching | **Cached**, later clicks may skip our server | Not cached, every click hits us |
| Server load | Lower | Higher |
| Analytics | **Clicks are lost** after the first one | Every click can be counted |
| Changing the target later | Hard (browsers remember) | Easy |

Say: "If we need click analytics or the ability to change or expire a link, use 302. If we only care about load and SEO, use 301."

## 9. Expiration and cleanup
- Store `expire_at`. On read, check it; expired links return 404 or 410.
- **Lazy deletion** (check on read) plus a **background job** to purge expired rows in batches.
- With TTL support in the store (for example, Redis or DynamoDB TTL), expiry can be automatic.

## 10. Analytics（不能拖慢跳转）
- Don't write analytics in the redirect path. Publish a click event (code, time, referrer, user agent) to **Kafka**, and process it asynchronously into counters or a data warehouse.
- Counters can be approximate. Aggregate in batches instead of one DB write per click.

## 11. Abuse, security, and reliability
- **Rate limit** creation per user or IP (see `rate-limiter-system-design-prep.md`).
- **Validate URLs**: scheme must be http or https, block malicious domains (malware and phishing lists), prevent redirect loops to our own domain.
- **Unguessable codes** if links should be private; otherwise sequential codes allow enumeration.
- **High availability**: stateless app servers behind a load balancer, replicated DB, multi-AZ. If the cache is down, fall back to the DB (and protect the DB with rate limiting).
- **Idempotent creation**: same long URL from the same user can return the existing code (look up a secondary index on `(user, hash(long_url))`).

## 12. Trade-offs to mention
- **Counter vs hash:** counter has no collisions but needs coordination and is guessable; hash is stateless but needs collision handling.
- **301 vs 302:** performance vs analytics and control.
- **SQL vs NoSQL:** the workload is simple key lookups at large scale, so a sharded key-value store fits well.
- **Consistency:** a newly created link must work immediately, so read-your-write matters. Write to the DB first, then populate the cache.

## 13. 30-second answer（口述版）
"First I clarify scale: it's read-heavy, around a hundred reads per write, so the cache matters most. For the code, I get a unique ID from a range allocator or a Snowflake-style generator, scramble it with a one-to-one mapping so it can't be guessed and can't collide, and encode it in base62. The mapping goes into a key-value store keyed by code, sharded by hash of the code. Redirects go through a Bloom filter and then Redis, and I use 302 so I can count clicks. Click events go through Kafka, off the redirect path. The Bloom filter protects the database from lookups of codes that don't exist."

## 14. 常见追问（自测）
1. 怎么保证短码唯一？多台服务器同时生成会不会冲突？
2. 为什么用 base62？7 位够用多久？
3. 哈希方案的碰撞怎么处理？
4. 301 和 302 怎么选？
5. 缓存穿透怎么防？（Bloom filter、缓存空值）
6. 热点链接怎么处理？
7. 怎么统计点击量，又不影响跳转速度？
8. 过期链接怎么清理？
9. 怎么防止被滥用（垃圾、钓鱼链接）？
10. 数据量变大后怎么分片？

## 15. 和你项目的连接
- **缓存设计**：FXCenter 的"读完全走本地缓存 + 预加载 + 完整性检查 + fallback"，和短链接"读走缓存、miss 才回源"同一类思路。
- **异步解耦**：点击事件进 Kafka，类似支付记账数据进 Kafka 再做 ETL。
- **幂等**：重复创建同一个长 URL，类似请求带 idempotency key。
- **限流**：创建接口用 rate limiter 保护。
