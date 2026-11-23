# API Design：可以向面试官提问的问题

用法：开头问 3–5 个最关键的问题（2 分钟内），然后说明假设，再开始设计。不要问太多，面试官会觉得你在拖时间。每个问题最好带一个"我的默认假设"，对方不回答你也能往下走。

开场句：
> "Before I start, I'd like to clarify a few things so I design the right thing."

结束句：
> "Here are my assumptions. Does this match what you had in mind?"

## 1. 最该问的 5 个（时间不够只问这些）

| # | 问题（中文） | English | 默认假设 |
| --- | --- | --- | --- |
| 1 | 谁调用这个接口？商户服务器、浏览器还是内部服务？ | Who calls this API: merchant servers, browsers, or internal services? | 商户服务器 |
| 2 | 这个接口的核心用例是什么？成功路径要返回什么？ | What is the main use case, and what should the caller get back? | 一次支付，返回状态 |
| 3 | 重复请求或超时重试会发生吗？ | Can the caller retry, and can duplicates happen? | 会，所以要幂等键 |
| 4 | 数据一致性要求有多高？可以接受"处理中"状态吗？ | How strong must consistency be? Is a pending state acceptable? | 钱要强一致，允许 PENDING |
| 5 | 什么不在范围内？（3DS、退款、风控、对账） | What is out of scope? (3DS, refunds, fraud, reconciliation) | 先不做 |

## 2. 按主题分组的问题

### 2.1 调用方和用例
- 有几类调用方？是否有第三方商户？（决定认证方式和 API 稳定性要求）
- 有没有已有的接口或前端要兼容？
- 同步返回结果还是异步通知（webhook）？
- English: "Is the response synchronous, or do clients get the final result through a webhook?"

### 2.2 请求和响应
- 必填和可选字段有哪些？金额的单位是什么（最小货币单位）？币种怎么表示（ISO 4217）？
- 成功和失败的响应格式有没有要求？
- 是否需要一次返回**多个**校验错误？
- English: "If the request has several problems, should I return all of them at once?"

### 2.3 校验和责任划分
- 哪些检查由我们做，哪些交给下游（银行、模拟器）？
- 请求无效时，要不要调下游？（一般不调）
- 下游的结果有哪几种？超时算什么？
- English: "Which checks should we do ourselves, and which should be left to the bank? And what do we return when the bank times out?"

### 2.4 幂等和重试
- 调用方会带幂等键吗？有效期多久？
- 同一个 key 不同请求体怎么办？（我默认返回 409）
- 我们调下游失败后，需要自动重试吗？
- English: "Will callers send an idempotency key? What should happen if the same key arrives with a different body?"

### 2.5 一致性和状态
- 有哪些状态？状态能不能回退？
- 超时后状态不确定，是否允许 PENDING，再由后台确认？
- 需要对账吗？
- English: "Is it acceptable to return a pending status when the bank is slow, and confirm later?"

### 2.6 安全
- 怎么认证调用方？（API key、OAuth2）
- 敏感数据（卡号、CVV）能不能进入我们的系统？还是只收 token？
- 日志里允许记录什么？
- 需要限流吗？
- English: "Can the API receive raw card numbers, or only tokens? That decides our PCI scope."

### 2.7 规模和性能
- 预期 QPS 和峰值？读多还是写多？
- 延迟要求（例如 p99 小于 1 秒）？
- 下游（银行）的限流和延迟是多少？
- English: "What traffic and latency should I plan for?"

### 2.8 版本、兼容和运维
- API 要不要版本号？会不会有旧客户端？
- 需要什么监控和告警？
- 部署后如何回滚？
- English: "Do we need to support multiple API versions?"

### 2.9 测试和交付（编码题用）
- 要不要写测试？用什么框架？
- 下游用模拟器还是 mock？
- 要不要持久化，还是内存即可？
- 有没有技术栈限制（Java、Spring）？
- English: "Should I persist data, or is in-memory fine for now? Are there any framework constraints?"

## 3. 按题型选问题

| 题型 | 重点问 |
| --- | --- |
| 设计一个支付接口 | 1、3、4、6（卡号是否进来）、2.3（校验和银行的责任划分） |
| 编码题（带银行模拟器） | 2.2、2.3、2.9：响应格式、多个错误、超时怎么办、要不要持久化 |
| 设计一个 webhook | 重试策略、签名、顺序、重复 |
| 设计一个查询或列表接口 | 分页、排序、过滤、数据量、缓存 |
| 设计退款或扣款类接口 | 幂等、状态机、部分退款、对账 |

## 4. 怎么问才好

1. **先问、后说假设**：问完立刻写在白板上，例如 "I'll assume: merchant servers call us, tokens only, retries are possible."
2. **问题要影响设计**：每个问题要有"答案不同，设计也不同"。例如"卡号进不进来"决定 PCI 范围。
3. **带默认值**："I'll assume X unless you say otherwise." 对方不回答也能往下走。
4. **别问文档里写了的**：先读题，题目里已有的信息不要再问。
5. **不要一次问十个**：先问 3–5 个，设计过程中再按需追问。
6. **说出范围外的东西**："I'll leave 3DS and fraud out for now. Is that okay?"
7. **结束时确认**："Does this match what you had in mind?"

## 5. 以这道题为例（Rejected / Authorized / Declined）

可以问的 5 个问题：

1. "Can a request contain several errors, and should I report all of them at once?"（多个错误一次返回）
2. "What should happen when the bank simulator times out: Declined, or something else?"（超时算什么）
3. "Which checks do you expect in the gateway, for example Luhn and expiry, and which should be left to the bank?"（责任划分）
4. "Can the caller retry? Should I support an idempotency key?"（幂等）
5. "Should I store the payment, or is in-memory fine for now?"（是否持久化）

然后说假设：
"I'll assume amounts are integers in minor units, currencies are ISO codes, card data arrives in the request for this exercise, and I'll never log or return the full card number."

## 6. 最后一条提醒

问问题是在展示你的思路，不是在等答案。**每问一个就说一句"这会影响我的设计，因为……"**，面试官会看到你在思考权衡。

例如："I'm asking about retries because that decides whether I need an idempotency key and a pending state."
