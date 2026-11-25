# Take-home 演示稿：Payment Gateway

用法：中文是给你看的提示，`>` 开头的英文是可以直接说的话。先讲、再演示、再讲代码，最后准备被追问。
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

- 浏览器打开 `http://localhost:8090/swagger-ui/index.html`，展开 POST /payment，点 **Try it out**。
- 备用方案：Swagger 打不开时，用 curl 手动发 `src/main/resources/openapi-examples/` 里的 JSON，例如 `curl -s -X POST localhost:8090/payment -H 'Content-Type: application/json' -d @src/main/resources/openapi-examples/1-authorized.json`。
- 开好这几个文件的标签页：`PaymentGatewayService`、`PaymentValidator`、`RestBankClient`、`CommonExceptionHandler`、`DESIGN.md`。
- 不要演示完整卡号的日志。需要看日志时只展示"没有卡号"这件事。

---

## 1. 开场（30 秒）

> I built a payment gateway that sits between a merchant and an acquiring bank. A merchant posts card details, I validate them, call the bank, and return Authorized, Declined or Rejected. I store only the last four digits, and the merchant can fetch the payment later by id. I'll show it working, walk through the code, then happy to talk about security and scale.

要点：先说**做了什么**，再说**会怎么讲**，让面试官知道结构。

---

## 2. 一张图讲清流程（1 分钟）

```
Merchant ──POST /payment──▶ Controller ──▶ Service ──▶ Validator   (不合法：422，不调用银行)
                                             │
                                             ├──▶ BankClient ──▶ Bank simulator  (503/超时：502，不保存)
                                             │
                                             └──▶ Repository (内存，只存后四位)
Merchant ──GET /payment/{id}──▶ Controller ──▶ Service ──▶ Repository
```

> The controller is thin. The service owns the flow: validate, call the bank, map the result, store. The bank is behind an interface, so the service can be tested without a network.

---

## 3. 现场演示（5 分钟，在 Swagger 里选下拉框示例，点 Execute）

看 **Server response** 里的 Code，不要看 Responses 表格。

| 步骤 | 选哪个示例 | 预期 | 要说的话 |
| --- | --- | --- | --- |
| 1 | 1. Authorized | 200，`Authorized`，只有后四位 | "The bank simulator authorizes cards ending in an odd digit. Notice the response has only the last four digits and an id." |
| 2 | 2. Declined | 200，`Declined` | "Declined is a normal business result, so it's still 200. The payment is stored either way." |
| 3 | 4. Several invalid fields | 422，`Rejected`，多个错误 | "All errors come back at once, so the merchant fixes everything in one round trip. The bank is never called." |
| 4 | 5. Fields missing | 422，每个必填字段一条 | "Missing fields are also content errors, because the JSON itself is valid." |
| 5 | 6. Card expired | 422，一个 expiry 错误 | "The expiry month itself is still valid. The clock is injected, so this is testable." |
| 6 | 8. Amount not a whole number | 422，`amount` 字段错误 | "Amount is in minor units, so 10.5 is rejected instead of silently rounded. Changing a money amount silently would be dangerous." |
| 7 | 7. Malformed JSON | 400 | "400 means I cannot read the request. 422 means I read it but the content is not acceptable." |
| 8 | 3. Bank unavailable | 502 | "The bank returned 503. I return 502 and store nothing, because I don't have a result to record." |
| 9 | GET /payment/{id} | 200，粘贴步骤 1 返回的 id | "Retrieving the payment returns the same data. Still only the last four digits." |

可选：GET 一个随机 UUID 返回 404；GET `not-a-uuid` 返回 400。

---

## 4. 代码走读（3 分钟，按这个顺序）

1. `PaymentGatewayService.processPayment`：validate → bank → map status → store。"This is the whole flow in 20 lines."
2. `PaymentValidator`：所有规则，收集全部错误，每个字段一条消息，`Clock` 注入。
3. `RestBankClient` + `BankClient` 接口：snake_case、`MM/YYYY`，503、超时、空响应都变成 `BankUnavailableException`。
4. `CommonExceptionHandler`：422 / 502 / 400 / 404 的映射，400 用固定文案。
5. `PostPaymentRequest`：`toString()` 脱敏。
6. `DESIGN.md`：假设、没做的、测试。

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

**再坦白没做到的（Not done, and what I'd add）：**

| 问题 | 回答要点 |
| --- | --- |
| 认证？ | 没做（brief 没要求）。生产里每个商户一个 API key，放在 `Authorization` header，保存时只存哈希。 |
| 越权？（任何人能 GET 任何 id） | 现在确实可以。UUID 随机难猜，**但不能当作授权**。生产里支付记录带 merchant id，查询按 merchant id 过滤。这是 IDOR 风险，要主动说出来。 |
| 传输加密？ | 代码里没有，TLS 在负载均衡或网关终止。 |
| 存储 PAN / CVV？ | PCI 规则：CVV 授权后不能存。完整卡号只能放在合规的 vault 里，网关只存 token 加后四位（对应 Round 3 的 Card Vault 设计）。 |
| 日志脱敏？ | 见上。再补一点：生产里加一个测试，往日志里打请求，断言没有卡号。 |
| 限流 / 防重放？ | 没做。API 网关限流；重放靠幂等 key。 |
| 输入攻击？ | 现在没有 SQL（内存存储）。上数据库后只用参数化查询。字段长度都有上限，卡号 14–19 位。 |
| Swagger UI 暴露？ | 演示用。生产里关掉，或放在内网。 |

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
