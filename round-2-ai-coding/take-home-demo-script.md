# Take-home 演示稿：Payment Gateway

用法：整场演示分三块，按这个顺序切换屏幕：**DESIGN.md（流程图）→ Swagger（演示）→ 代码（走读）**。
中文是给你看的提示，`>` 开头的英文是可以直接说的话。最后留时间被追问。
时间 [TODO 向 recruiter 确认面试时长和形式]，下面按 15 分钟讲 + 追问来排。

---

## 0. 演示前检查（5 分钟前做完）

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17 && export PATH=$JAVA_HOME/bin:$PATH
cd /Users/lintao/codes/payment-gateway-challenge-java
docker-compose up -d      # 银行模拟器 :8080（Docker 没起就先打开 colima / Docker Desktop）
./gradlew test            # 确认 79 个测试全绿
./gradlew bootRun         # 网关 :8090
```

- 标签页 1，**DESIGN.md**：VS Code 打开 `DESIGN.md` 的 Markdown 预览，确认 "Request flow" 的 Mermaid 图能渲染。
- 标签页 2，**Swagger**：浏览器打开 `http://localhost:8090/swagger-ui/index.html`，展开 POST /payment，点 **Try it out**。
- 标签页 3，**代码**：`PaymentGatewayService`、`PaymentValidator`、`RestBankClient`、`CommonExceptionHandler`、`DESIGN.md`。
- 备用方案：Swagger 打不开时，用 curl 发 `src/main/resources/openapi-examples/` 里的 JSON，例如 `curl -s -X POST localhost:8090/payment -H 'Content-Type: application/json' -d @src/main/resources/openapi-examples/1-authorized.json`。
- 不要演示完整卡号的日志。需要看日志时只展示"没有卡号"这件事。

---

## 1. 开场（30 秒）

> For the take-home, the task was to implement a payment gateway API: a merchant posts card details, the gateway validates them, forwards them to an acquiring bank simulator, and returns the result. The merchant can also retrieve a payment by id. I'll start with a quick demo, then walk through the design decisions and my assumptions, and I'm happy to go deeper on security and scalability.

要点：
- 这是**作业**，不要说成"我自己做的产品"。先说**任务是什么**，再说**会怎么讲**，让面试官知道结构。
- 说"我做了哪些设计决定和假设"，比说"我 build 了"更准确。
- 如果问到 AI 的使用，按事实回答（见 §8）。

---

## 2. 【DESIGN.md】一张图讲清流程（1 分钟）

切到 DESIGN.md 开头的 "Request flow" 图。沿着实线讲一遍，再指虚线。

> The controller is thin. The service owns the flow: validate, call the bank, map the result, store. The solid arrows are the normal path. The dotted arrows are failures: invalid content is a 422 and the bank is never called, a bank 503 or timeout is a 502 and nothing is stored, and unreadable JSON is a 400. The bank is behind an interface, so the service can be tested without a network.

同一份 DESIGN.md 里往下就是假设和取舍（"Not done on purpose" 表），后面被问到直接往下翻，不用切换文件。

---

## 3. 【Swagger】现场演示（5 分钟，选下拉框示例，点 Execute）

切到 Swagger。看 **Server response** 里的 Code，不要看 Responses 表格（那里列的是文档里声明的状态码）。

开场说一句：> The examples are pre-filled in the drop-down, so I just pick one and click Execute.

| 步骤 | 选哪个示例（下拉框顺序和这里一致） | 预期 | 要说的话 |
| --- | --- | --- | --- |
| 1 | 1. Authorized | 200，`Authorized`，只有后四位 | "The bank simulator authorizes cards ending in an odd digit. Notice the response has only the last four digits and an id." |
| 2 | 2. Declined | 200，`Declined` | "Declined is a normal business result, so it's still 200. The payment is stored either way." |
| 3 | 3. Several invalid fields | 422，`Rejected`，多个错误 | "All errors come back at once, so the merchant fixes everything in one round trip. The bank is never called." |
| 4 | 4. Fields missing | 422，每个必填字段一条 | "Missing fields are also content errors, because the JSON itself is valid." |
| 5 | 5. Card expired | 422，一个 expiry 错误 | "The expiry month itself is still valid. The clock is injected, so this is testable." |
| 6 | 6. Amount not a whole number | 422，`amount` 字段错误 | "Amount is in minor units, so 10.5 is rejected instead of silently rounded. Changing a money amount silently would be dangerous." |
| 7 | 7. Malformed JSON | 400 | "400 means I cannot read the request. 422 means I read it but the content is not acceptable." |
| 8 | 8. Bank unavailable | 502 | "The bank returned 503. I return 502 and store nothing, because I don't have a result to record." |
| 9 | GET /payment/{id} | 200，粘贴步骤 1 返回的 id | "Retrieving the payment returns the same data. Still only the last four digits." |

可选：GET 一个随机 UUID 返回 404；GET `not-a-uuid` 返回 400。

---

## 4. 【代码】走读（3 分钟，按这个顺序）

演示里每个状态码，都对应到一处代码：

1. `PaymentGatewayService.processPayment`：validate → bank → map status → store。"This is the whole flow in 20 lines, the same as the diagram."
2. `PaymentValidator`：对应 422。所有规则，收集全部错误，每个字段一条消息，`Clock` 注入。
3. `RestBankClient` + `BankClient` 接口：对应 502。snake_case、`MM/YYYY`，503、超时、空响应都变成 `BankUnavailableException`。
4. `CommonExceptionHandler`：422 / 502 / 400 / 404 的映射，400 用固定文案。
5. `PostPaymentRequest`：`toString()` 脱敏，对应"没有卡号泄露"。
6. `OpenApiConfiguration` + `resources/openapi-examples/`：Swagger 的示例从这里来，控制器里没有文档注解。
7. `DESIGN.md`：假设、没做的、测试。

---

## 5. 关键决定（Decision → Reason → Trade-off）

| 决定 | 原因 | 代价 |
| --- | --- | --- |
| 内容不合法返回 422 并一次列出所有错误；读不懂返回 400 | 商户一次就能改完；400 和 422 的修复方式不同 | 调用方要区分两种错误 |
| 银行不可用返回 502，不保存 | 没有结果可记录，不应该保存一个不确定的支付 | 超时时商户不知道银行有没有扣款（见 §8） |
| 金额用最小单位整数（`BigDecimal` 读入，校验后转 `long`） | 钱不用浮点数；10.5 不会被悄悄截断 | 不处理各币种的小数位差异（JPY 没有小数） |
| 币种接受所有 ISO 4217（`java.util.Currency`） | README 的"不超过 3 个币种"我理解为上限而不是固定列表。这是**我的理解，已写入 DESIGN.md** | `XXX` 等特殊代码也会被接受 |
| 不做 Luhn 校验 | 模拟器的测试卡不一定通过 Luhn，brief 也没要求 | 少一层明显错误的拦截 |
| 银行放在接口后面 | 服务层可以用假的银行测试，换银行不用改服务 | 多一个接口 |
| 严格 JSON（`"100"` 不会被转成数字） | 避免悄悄改值 | 对调用方更严格 |

---

## 6. 安全性（面试官一定会问）

**先说已经做到的（Already done）：**

> I never store, log, or return the full card number or CVV. Only the last four digits are kept. The request record's `toString()` is redacted, validation messages describe the rule and never quote the submitted value, and the 400 message is fixed text because Jackson's own message can quote the input. I checked the application log after running against the real simulator, and there was no card data.

**再坦白没做到的（Not done）。先用一句话开头，再逐个讲，每个都是 问题 → 方案 → 代价：**

> There are a few security gaps I know about. I kept them out of scope because the brief didn't ask for them, but I can tell you how I would fix each one. The two most important are authentication and per-merchant access.

### 1. Authentication（没有认证）
> **Problem:** Anyone who can reach the API can create payments.
> **Fix:** Give each merchant an API key and send it in the `Authorization` header. A filter checks the key before the request reaches the controller. I store only a hash of the key, like a password.
> **Trade-off:** Keys must be rotated and revoked, so I need a small key-management flow.

#### 1.1 API key flow 拆解（被追问"具体怎么做"时讲）

**A. 发放 key（Issue）**
1. 商户注册后，我们生成一个**随机的长 key**（例如 32 字节随机数，带前缀，如 `sk_live_...`）。
2. 只在创建时给商户**看一次**。
3. 数据库里存：`key_id`（key 的前缀，用来查找）、**key 的哈希**、`merchant_id`、状态（active / revoked）、创建时间、过期时间。**不存原始 key。**

> We generate a long random key, show it to the merchant once, and store only its hash, plus a short key id, the merchant id and a status.

**B. 每个请求怎么验证（Verify）**
1. 商户在 header 里带上 `Authorization: Bearer <key>`。
2. 一个 filter 在 Controller 之前运行：取出 key，用前缀找到那一行，对收到的 key 做哈希，和库里的哈希比较（用**常量时间比较**）。
3. 状态必须是 active，并且没过期。
4. 通过后，把 `merchant_id` 放进请求上下文。后面的 Service 用它，这正是 §2 里"按商户过滤"所需要的。
5. 没带 key、key 错误或已撤销，返回 **401**，不进入 Controller。

> A filter runs before the controller. It finds the key by its id, hashes the incoming key, and compares it with the stored hash. If it matches and the key is active, the request continues with the merchant id attached. If not, we return 401.

**C. 轮换（Rotate）**
1. 商户创建**第二个** key，两个同时有效一段时间。
2. 商户把系统切换到新 key。
3. 旧 key 设为 revoked。

> Rotation works with two active keys at the same time, so the merchant can switch with no downtime.

**D. 撤销（Revoke）**
1. key 泄露时，商户或我们把状态改成 revoked。
2. 下一个请求立刻被拒绝。如果验证结果有缓存，缓存时间要很短（例如 1 分钟），或者撤销时主动清掉缓存。

> If a key leaks, we mark it revoked and the next request is rejected.

**E. 为什么这样设计（可能的追问）**

| 追问 | 回答 |
| --- | --- |
| 为什么存哈希，不存原始 key？ | 数据库泄露时，攻击者拿到的哈希不能直接用来调用 API。 |
| 为什么不用 bcrypt，用 SHA-256 就行？ | key 是**长随机数**，不是用户选的密码，不能被字典攻击猜出来。SHA-256 又快，每个请求都要验证，用 bcrypt 会拖慢。这个理由要主动说出来。 |
| 为什么用前缀查找？ | 先用 `key_id` 精确找到一行，再比较哈希，避免遍历所有 key。 |
| 为什么用常量时间比较？ | 普通比较遇到第一个不同的字符就停，攻击者可以通过响应时间一点点猜出 key。 |
| 401 还是 403？ | 401：你是谁我不知道（没带或错误的 key）。403：我知道你是谁，但你不能做这件事。 |
| key 放在 URL 里行吗？ | 不行，URL 会被写进日志和浏览器历史。放 header。 |
| 日志里要注意什么？ | 绝不打印 key，和卡号一样要脱敏。 |
| 怎么实现？ | Spring 里用 `OncePerRequestFilter`，或者用 Spring Security 的自定义认证过滤器。小项目先用简单的 filter 就够。 |
| 更强的方案？ | 请求签名（HMAC）、mTLS，或 OAuth2 客户端凭证。API key 简单，适合第一版。 |
| 这和 IDOR 的关系？ | 认证回答"你是谁"，得到 `merchant_id`。授权（§2）用它回答"你能不能看这笔支付"，两者要一起做。 |

**一句话总结：**
> I'd use per-merchant API keys: random, shown once, stored as a hash, checked in a filter before the controller. That gives me a merchant id for every request, and I use it to scope all data access. Keys can be rotated with two active keys and revoked instantly.

#### 1.2 怎么防止 API key 被别人盗用？（被追问时讲）

**先说实话：** 没有办法 100% 保证 key 不被偷，所以用**多层防御**：让它难被偷、被偷后伤害小、被偷后能很快发现并止损。

> I can't guarantee a key is never stolen. So I use layers: make it hard to steal, limit the damage if it is stolen, detect it quickly, and be able to revoke it fast.

**第 1 层：让它难被偷（Prevent）**

| 方面 | 做法 |
| --- | --- |
| 传输 | 只允许 HTTPS，别人在网络上抓不到 key |
| 我们这边的存储 | 只存哈希，数据库泄露也拿不到可用的 key |
| 日志 | 绝不打印 key，header 要脱敏 |
| 商户这边 | 告诉商户：key 只能放在**服务端**，放在密钥管理服务或环境变量里，不能放在手机 App 或前端代码里，也不能提交到 Git |
| 强度 | key 是足够长的随机数，不能被猜出来 |

> Keys travel only over HTTPS, we store only hashes, we never log them, and we tell merchants to keep them on the server side, never in a mobile app, browser code or Git.

**第 2 层：被偷后伤害小（Limit the damage）**

| 做法 | 说明 |
| --- | --- |
| 最小权限（scope） | 例如"只读 key"只能 GET，"支付 key"才能 POST |
| IP 白名单 | 商户可以限制 key 只能从自己服务器的 IP 调用 |
| 按 key 限流和限额 | 即使被偷，也不能无限制地调用或刷大额 |
| 测试 key 和正式 key 分开 | 测试环境泄露不影响真实资金 |
| 有效期 | key 定期过期，强制轮换 |

> Each key has limited scope, an optional IP allow-list, its own rate limit, and an expiry date. A stolen key can only do so much.

**第 3 层：尽快发现（Detect）**
- 监控异常：突然来自新的 IP 或国家、请求量暴增、大量 Declined（可能在测试被盗卡号）。
- 异常时告警，必要时自动暂停该 key。
- 对每个 key 记录最后使用时间和来源，商户自己也能在后台看到。
- 在 GitHub 上开启 **secret scanning**，key 带固定前缀（例如 `sk_live_`），这样泄露到公开代码里时能被自动发现。

> We watch for unusual usage, like a new IP, a traffic spike, or many declines, and alert on it. Keys have a fixed prefix so secret scanners can spot them if they leak into public code.

**第 4 层：快速止损（Respond）**
- 商户或我们可以立刻**撤销**这个 key。
- 轮换流程已经有（新旧两个 key 同时有效），所以换 key 不会停机。
- 有事件处理流程：撤销、通知商户、检查期间的交易。

**更强的方案（如果面试官想要更高安全）：**
- **请求签名（HMAC）**：商户用 secret 对每个请求签名，secret 本身不在网络上传输，还能带时间戳防重放。
- **mTLS**：双向证书，服务器验证客户端证书。
- **OAuth2 客户端凭证**：令牌短期有效，泄露后很快失效。
- 取舍：都比 API key 复杂，商户接入成本更高，所以第一版用 API key，之后再升级。

**一句话总结：**
> No single control is enough. I'd combine HTTPS and hashing to protect the key, scopes and IP limits to reduce the damage, monitoring to spot misuse, and fast revoke and rotate to recover. If we need stronger security later, I'd move to request signing.

### 2. Authorization / IDOR（任何人能读任何支付）
> **Problem:** Today, anyone who knows a payment id can read that payment. A random UUID is hard to guess, but that is not real access control.
> **Fix:** Save the merchant id with every payment. On `GET`, look up by payment id **and** merchant id. If it is not theirs, return 404, not 403, so we don't reveal that the payment exists.
> **Trade-off:** One more column and one more condition in every query. It is cheap.

### 3. Encryption in transit（传输加密）
> **Problem:** The app itself speaks plain HTTP.
> **Fix:** Terminate TLS at the load balancer or API gateway. Use TLS for the call to the bank too. Never accept plain HTTP in production.
> **Trade-off:** Certificates need renewal, but that is usually automated.

### 4. Card data at rest（卡号和 CVV 的存储）
> **Problem:** If we ever stored the full card number, a database leak would be a disaster, and PCI rules are strict.
> **Fix:** Today we keep only the last four digits, and we never keep the CVV. If the business needs the full number later, put it in a separate, PCI-compliant card vault, encrypted, and give the gateway only a token.
> **Trade-off:** A vault is extra infrastructure, so I would not build it unless we need repeat payments.

### 5. Rate limiting and abuse（限流和滥用）
> **Problem:** A stolen key or a bot could send many requests. Attackers also use a payment API to test stolen card numbers (card testing).
> **Fix:** Rate limit per merchant at the API gateway, and watch for many Declined results from one merchant. Add a request size limit so huge bodies are rejected early.
> **Trade-off:** A limit that is too low blocks real traffic, so I would start loose and tune with real data.

### 6. Replay and duplicate requests（重放和重复请求）
> **Problem:** A retry, or a replayed request, creates a second payment.
> **Fix:** Require an `Idempotency-Key` header. Store it with a unique constraint. The same key returns the first result instead of charging again.
> **Trade-off:** One extra write per request, and we must decide how long to keep keys, for example 24 hours.

### 7. Secrets and configuration（密钥和配置）
> **Problem:** Today the bank URL is in `application.properties`. Later we will have API keys and bank credentials.
> **Fix:** Load secrets from a secret manager or environment variables. Never commit them to Git.
> **Trade-off:** Local setup is a bit harder.

### 8. Logging, monitoring and audit（日志、监控、审计）
> **Problem:** We log no card data, which is good, but we have no automatic check for it, no request id and no alerts.
> **Fix:** Add a test that fails if a card number appears in logs. Add a request id to every log line. Alert on a high bank error rate and a high Rejected rate. Keep an audit log of who did what, without sensitive data.
> **Trade-off:** More logs cost more to store, so keep them structured and short.

### 9. Swagger UI and error details（Swagger 和错误信息）
> **Problem:** Swagger UI is useful for the demo, but it shows the API to everyone.
> **Fix:** Turn it off in production, or keep it behind the internal network. Error messages already don't repeat what the user sent, and the 400 message is fixed text.
> **Trade-off:** None really, it is a config switch per environment.

### 10. Dependencies（依赖安全）
> **Problem:** A library with a known vulnerability puts the service at risk.
> **Fix:** Run a dependency scan in CI and update regularly.
> **Trade-off:** Updates sometimes break things, so tests matter.

**被问"先修哪个？"：**
> I would fix authentication and merchant-level access first, because without them anyone can read or create payments. Then idempotency, because duplicate charges hurt real money. Then rate limiting and monitoring.

**被问"你怎么证明没有泄露卡号？"：**
> Three things. The request's `toString()` never prints the card number or CVV. Error messages describe the rule and never quote the value. And I ran the app against the simulator and searched the log for card digits and found none. One gap: that last check was manual, so next I would add an automated test.

---

## 7. 可扩展性（面试官一定会问）

**先问澄清问题（clarify first）：**
> What traffic should I assume, and is it write-heavy? A payment gateway is write-heavy on POST and read-light on GET.

**回答结构：**

1. **现在的瓶颈和限制**
   - 内存 `ConcurrentHashMap`：线程安全，但数据在单个进程里，**不能水平扩展，重启就丢**。这是第一个要换的东西。
   - 同步调用银行，超时是 10 秒（`RestTemplate` 的 connect 和 read）。流量大、银行慢时，线程会被占满。
2. **我会怎么扩（按顺序，KISS）：**
   - 数据库（有 payment id 索引），服务变成无状态，多实例放在负载均衡后面。
   - 银行调用：超时调短，加熔断器和舱壁隔离（bulkhead），一个银行变慢不能拖垮整个网关。
   - 幂等表（`Idempotency-Key` 加唯一约束）。
   - 读：GET 走缓存或只读副本，写走主库。
   - 数据增长：按 merchant id 分区，历史数据归档。
   - 非关键工作异步化：webhook、对账、审计事件。
3. **Trade-off：** 同步等银行结果，延迟高，但商户马上得到答案；异步加 webhook 吞吐更好，但商户要处理"待处理"状态，复杂度更高。

**10 倍流量时什么先坏？** 银行调用的线程耗尽，其次才是数据库。

---

## 8. 其他可能的追问

| 追问 | 回答 |
| --- | --- |
| **银行超时了，怎么知道有没有扣款？** | 现在不知道，这是已知限制（DESIGN.md 里写了）。生产：调用前先保存 `PENDING`，超时后按 payment id 向银行查询，再用 webhook 通知商户。 |
| **商户重试会怎样？** | 会创建第二笔支付（有一个测试记录了这个行为）。生产里用 `Idempotency-Key` 加唯一约束，并发重试也只产生一笔。 |
| **并发？** | POST 只是插入，没有共享的计数器或余额，所以没有更新丢失问题。`ConcurrentHashMap` 保证并发读写安全。要更新余额才需要锁或版本号（对应之前的悲观锁/乐观锁讨论）。 |
| **为什么 amount 不是整数是 422，不是 400？** | 分层规则：读不懂是 400，读得懂但内容不合法是 422。`10.5` 是数字，只是不符合"整数最小单位"这条业务规则。字符串 `"100"` 是类型错误，所以是 400。 |
| **为什么不是 200 返回 Declined 以外的状态码？** | Declined 是银行给出的正常业务结果，请求本身是成功处理的。 |
| **日志里有什么？** | 支付 id 和状态，没有卡号。生产里加 request id、指标（成功率、延迟、错误数）和告警。 |
| **怎么测的？** | 79 个测试：校验器每条规则和边界；银行客户端用 `MockRestServiceServer`；服务层用假的银行；控制器用完整 Spring 上下文。另外对真实的 Docker 模拟器跑过三种结果。 |
| **你用了 AI 吗？** | [TODO 按事实回答。如果用了：说明你审查了每一步，做了哪些修改和决定（例如币种理解、422/400 规则、amount 的处理）。] |
| **如果重做会改什么？** | 先存 `PENDING` 再调银行；加幂等；数据库；商户认证。 |

---

## 9. 收尾（30 秒）

> To summarise: the happy path and all error paths are tested, no card data leaks, and the main gaps are idempotency, authentication, and persistence, which I've documented with a plan in DESIGN.md. Does this match what you were looking for?

---

## 10. 可以反问面试官

- 你们真实的网关怎么处理银行超时后的状态未知？
- 幂等 key 的范围是按商户，还是全局？
- 你们怎么满足 PCI 范围的要求？

---

## 11. 回答的节奏（提醒自己）

- 先**澄清问题**，边想边说，说出**不在范围内**的东西。
- 用 **Decision → Reason → Trade-off**。
- 不确定的事实不要编，说"I'm not sure, but I would check…"。
- 说出缺点比被问出来更好：认证、IDOR、幂等、超时状态未知，都要自己先提。
