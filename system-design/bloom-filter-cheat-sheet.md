# Bloom Filter – Cheat Sheet

## One-liner
> "A Bloom filter is a compact probabilistic set. If it says 'not present', the item is definitely not there. If it says 'present', it might be a false positive. I use it to cheaply reject lookups for keys that don't exist."
>
> 布隆过滤器：用很少的内存判断"一个元素是否在集合里"。**说不存在，一定不存在；说存在，可能是误判。** 不会漏报，会误报。

## 1. 它解决什么问题
- 集合很大，不想把所有元素都存起来，或者查询的代价很高（访问数据库、磁盘、远程服务）。
- 先问 Bloom filter：**一定不存在就直接返回，省掉昂贵的查询。**

## 2. 原理（三步理解）
- 一个长度为 **m** 的**位数组**（bit array），初始全是 0。
- **k 个**不同的哈希函数。
- **add(x)**：用 k 个哈希函数算出 k 个位置，把这 k 个位置都置为 1。
- **contains(x)**：算出同样的 k 个位置。
  - **只要有一个位置是 0** → **一定不存在**。
  - **全是 1** → **可能存在**（这些位置可能是别的元素置的）。

```
m = 10, k = 3

add("a") → 位置 1, 4, 7   位数组: 0 1 0 0 1 0 0 1 0 0
add("b") → 位置 2, 4, 9   位数组: 0 1 1 0 1 0 0 1 0 1
contains("c") → 位置 1, 2, 5
    位置 5 是 0 → 一定不存在
contains("d") → 位置 1, 2, 9
    全是 1 → "可能存在"，但 d 从没加过 → 这就是误判（false positive）
```

## 3. 特点（面试要点）
| 特点 | 说明 |
|---|---|
| **没有漏报（no false negatives）** | 加入过的元素，查询一定返回"可能存在" |
| **有误报（false positives）** | 没加入过的元素，可能被判为"可能存在" |
| **空间极小** | 每个元素大约 10 bit 就能做到约 1% 误判率 |
| **查询快** | O(k)，k 是常数 |
| **不能删除** | 一个位可能被多个元素共用，清零会影响别的元素 |
| **不能取出元素** | 只能判断，不存数据 |
| **容量不能随意扩** | 超过设计容量，误判率快速上升；扩容要重建 |

## 4. 参数怎么选
- **n**：预计元素个数；**p**：可接受的误判率；**m**：位数组大小；**k**：哈希函数个数。
- 最优公式（记住大意即可）：
  - `m = -n × ln(p) / (ln 2)²`，约等于 **每个元素 9.6 bit（p = 1%）**
  - `k = (m / n) × ln 2`，约等于 **7 个哈希函数（p = 1%）**
- 简单记法：**误判率要 1%，每个元素约 10 bit、7 个哈希。**
- 实际使用时直接给库传入 `n` 和 `p`，它自己算。

```java
// Guava
BloomFilter<String> filter =
    BloomFilter.create(Funnels.stringFunnel(StandardCharsets.UTF_8), 1_000_000, 0.01);
filter.put("abc");
filter.mightContain("abc");   // true
filter.mightContain("xyz");   // 大概率 false
```

## 5. 典型应用场景

**1. 防缓存穿透（最常考）**
- 问题：请求大量**不存在的 key**（比如恶意请求、不存在的短链接 code），缓存查不到，每次都打到数据库。
- 方案：把**所有存在的 key** 预先放入 Bloom filter。请求先问它：
  - 返回"不存在" → **直接返回，不查缓存和数据库**
  - 返回"可能存在" → 再走正常的缓存和数据库流程（误判的少量请求会多查一次库，可以接受）
- 对比：**缓存空值**（也能防穿透，简单，但大量不同的无效 key 会占很多缓存空间）。Bloom filter 更省内存。

**2. 爬虫 / 去重**
- 判断 URL 是否已经爬过，不用把所有 URL 存下来。

**3. 数据库 / 存储引擎**
- LSM 树（Cassandra、HBase、RocksDB）：读之前先用 Bloom filter 判断 key 是否可能在某个 SSTable 里，**避免无谓的磁盘读**。

**4. 垃圾邮件、黑名单、推荐去重**
- 判断"是否已经推荐/见过"，允许少量误判。

**5. 短链接**
- 请求的 code 不在 Bloom filter 里，直接 404，保护数据库。（见 `url-shortener-system-design-prep.md`）

## 6. 防缓存穿透的完整流程
```
请求 key
  → Bloom filter：一定不存在？ → 是 → 直接返回 404 / 空
  → 否（可能存在） → 查 Redis
        命中 → 返回
        未命中 → 查 DB
              有 → 写缓存，返回
              无（误判） → 可以缓存一个短 TTL 的空值，返回
```
**新增数据时**：写库成功后，也要把 key 加入 Bloom filter（否则新数据会被"误杀"成不存在，这是**漏报**，不允许）。

## 7. 缺点与应对

| 缺点 | 应对 |
|---|---|
| **不能删除元素** | **Counting Bloom Filter**（用计数器代替位，可以删除，但更占空间）；或者定期**重建**；Cuckoo Filter 支持删除 |
| **误判率随元素增多上升** | 按预计数量预留空间；超过容量时重建或使用 **Scalable Bloom Filter** |
| **数据变更要同步** | 新增数据要加入；删除的数据留在里面只会增加误判，不会造成错误 |
| **分布式共享** | 可以放在 Redis（`BF.ADD` / `BF.EXISTS`，RedisBloom 模块），或用 Redis 的 bitmap（`SETBIT` / `GETBIT`）自己实现；多实例各自加载同一份数据 |
| **启动加载** | 应用启动时从数据库全量加载 key 重建，或定时同步 |

## 8. 对比其他方案
| 方案 | 优点 | 缺点 |
|---|---|---|
| **HashSet** | 精确 | 内存大 |
| **Bitmap** | 精确，快 | key 必须是有限范围的整数，空间随范围增长 |
| **Bloom filter** | 内存很小，不漏报 | 有误报，不能删除 |
| **缓存空值** | 简单 | 大量无效 key 占缓存 |

## 9. 30-second answer（口述版）
"A Bloom filter is a bit array with several hash functions. To add an item, I set the bits at its hash positions. To check, if any of those bits is zero, the item is definitely not present. If all are one, it might be present, so there can be false positives but never false negatives. It's tiny, around 10 bits per item for a 1% error rate. I use it in front of a cache to stop cache penetration: lookups for keys that don't exist are rejected right away instead of hitting the database. The main limits are that it can't delete items and the error rate grows as it fills up, so I size it for the expected number of items or rebuild it periodically."

## 10. 常见追问（自测）
1. Bloom filter 的原理？为什么会误判，为什么不会漏报？
2. 能不能删除元素？怎么解决？
3. 误判率和什么有关？怎么设置参数？
4. 怎么用它防缓存穿透？新数据怎么处理？
5. 和缓存空值、HashSet 比有什么区别？
6. 分布式环境下怎么用？放在哪里？
7. 缓存穿透、击穿、雪崩的区别？（见 Redis 文档）

## 11. 和你项目的连接
- **FXCenter 缓存**：客户端缓存里查不到的币种对，如果是无效请求，可以在入口先拦截（取决于你们的实际设计，[待补]）。
- **幂等**：如果要判断"这个 requestId 是否处理过"，Bloom filter 只能做**粗筛**（说没处理过就一定没处理过），说"处理过"仍然要查库确认，因为有误判。**不能单独依赖它做幂等**。
