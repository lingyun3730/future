# Design: Payment Gateway for Thousands of Merchants（含 Card Vault 和 Risk Service，完整版）

> 题目：Design the gateway for thousands of merchants: where card data is stored, how you keep only the last four digits visible, and how a payment's status stays correct across retries.

**这是第 3 轮 system design 最可能的题。这一份文件把 Card Vault、Payment Service、Bank Adapter、Risk Service 放在一起，通篇准备。**

说明：这是通用的行业做法，不是我们公司的实际设计。面试时说 "I'd design it this way"，不要说成"我们做过"。更详细的支付流程、状态机、对账见 `payment-processor-design-prep.md`；API 校验和拆任务见 `payment-endpoint-validation.md`。

## 0. 怎么讲：一条主线，两个部分

```text
Part A 卡数据（安全）：卡号只在 vault -> 其他系统只拿 token + last4 -> 只有 Bank Adapter 能还原
Part B 支付状态（正确性）：幂等键 -> 先落库 PENDING -> 风控 -> 调银行 -> 条件更新 -> 恢复 worker -> 对账
两部分的连接点：支付请求里只有 token；风控只看脱敏特征；重试时只带 token，从不需要卡号
```

建议的讲述顺序（约 5 分钟）：Clarify（1 分钟）-> 架构总览（1 分钟）-> Part A（1.5 分钟，**只讲 §0.5 简化版**）-> Part B 含风控（1.5 分钟）-> 取舍和扩展（30 秒）-> 确认。

## 0.5 Vault 简化版（面试先讲这个，约 1 分钟；下面 §2、§3 是被追问时才用的完整版）

**一句话：卡号只存在 vault 里。其他系统只拿 token 和后四位。**

**只记 4 步：**

```text
(1) 浏览器 ──卡号──▶ Vault ──token──▶ 浏览器      收卡：卡号只走这一条路
(2) 商户服务器 ──token + 金额──▶ 我们的 Payment Service   商户只传 token
(3) Payment Service 只存 token + 后四位              我们的库里没有卡号
(4) Bank Adapter ──token──▶ Vault 换回卡号 ──▶ 银行   只有这一步能拿到卡号，用完就丢
```

**只讲 3 层保护（回答"卡号存哪、怎么只显示后四位"）：**

| 层 | 一句话 | English |
| --- | --- | --- |
| 1. 存 | 卡号加密后存在单独的 vault，密钥放在 KMS，不和数据放一起 | "Card numbers are encrypted in a separate vault, and the key lives in a key management service." |
| 2. 用 | 只有 Bank Adapter 能把 token 换回卡号，每次都记审计日志 | "Only the bank adapter can turn a token back into a card number, and every request is logged." |
| 3. 看 | 入库时后四位单独存一列，展示只读这一列；接口只返回白名单字段；日志脱敏 | "The last four digits are stored in their own column, so the rest of the system never needs the full number." |

**先不要讲（被追问再展开，细节在下面的 §3）：**
信封加密（DEK / KEK）、一次性 token 绑定支付、卡指纹、密钥轮换、vault 分片、PCI 范围。

**最容易被追问的 4 个问题，一句话答：**

| 问题 | 答案 | English（简短版） |
| --- | --- | --- |
| token 和加密有什么区别？ | token 是随机编号，和卡号没有数学关系，只能查 vault；加密的密文有密钥就能解开 | "A token is a random ID with no link to the card number, so you can only look it up in the vault. Encrypted data can be decrypted with the key." |
| token 被偷了怎么办？ | 单独没用：它绑定商户，只有 Bank Adapter 能还原，一次性 token 很快过期 | "A stolen token is not useful alone. It belongs to one merchant, only the bank adapter can turn it back into a card number, and one-time tokens expire quickly." |
| vault 挂了怎么办？ | 不绕过。返回可重试的 503，商户用同一个幂等键重试 | "We don't bypass it. We return a retryable 503, and the merchant retries with the same idempotency key." |
| 为什么卡号不经过 Payment Service？ | 这样只有 vault 和 Bank Adapter 需要通过最严格的合规审计，范围越小越好 | "So only the vault and the bank adapter are in the strictest compliance scope. A smaller scope is easier and safer." |
| BIN 是什么？ | 卡号前 6 位，表示发卡银行、卡组织、卡类型 | "The BIN is the first six digits of the card. It tells us the issuing bank and card type, so we can use it for risk checks without the full number." |
| 为什么能显示后四位？ | 信息量小，无法还原卡号，合规允许；单独一列，展示不需要解密 | "The last four digits can't rebuild the card number, and we store them in their own column, so showing them never needs decryption." |

**1 分钟英文口述：**
> "Card numbers live only in a vault. The browser sends the card number straight to the vault and gets back a token, so the merchant server never sees it. Our payment service stores only the token and the last four digits. When we call the bank, the bank adapter swaps the token for the card number, uses it once, and drops it. Only the adapter is allowed to do that, and every request is logged. The last four digits are stored in a separate column, so showing them never needs decryption. The trade-off is one extra call to the vault for each bank request."

---

## 1. Clarify 和假设

先问（至多 5 个，每个带默认假设）：

| # | 问题 | English | 默认假设 |
| --- | --- | --- | --- |
| 1 | 卡号由谁收集？商户服务器还是浏览器？ | Who sends the card data: the merchant's server or the browser? | 浏览器通过 hosted fields 直接发给 vault，商户请求里只有 token |
| 2 | 规模：峰值 TPS？ | What traffic should I plan for? | 平均几百 TPS，峰值约 1–2k（白板上写明，可调整） |
| 3 | 要支持存卡（以后再扣）吗？ | Do we need stored cards for repeat payments? | 要（订阅、复购） |
| 4 | 风控是现成的服务，还是我也要设计？ | Is there an existing risk service, or should I design it? | 现成的服务，我只设计集成方式 |
| 5 | 范围外？ | What is out of scope? | 3DS、结算、对账细节、退款先不展开，但会说明位置 |

确认句：`"I'll assume X, Y, Z, and leave 3DS and settlement out. Does that match what you had in mind?"`

**估算（说出来，不用精确）：**
- 峰值 1–2k TPS，每笔支付约 1KB，每笔至少 2 次写（插入 PENDING、更新结果）。
- 按平均 200 TPS 算，一年约 60 亿笔，每笔约 1KB，就是每年约 6TB，单库放不下也扛不住写入，所以**从一开始就按 transaction id 设计分片**（id 里带分片号），但流量没到之前不真正拆库。
- 每次调银行前要调一次 vault 还原，所以 vault 的请求量和调银行同量级，是**关键路径**。
- 存卡量远小于支付量，增长慢。

## 2. 总体架构

### 2.1 角色

```text
[Browser]          用户浏览器，页面里嵌着 vault 提供的输入框（hosted fields，iframe）
[Merchant Server]  商户后端
[API Gateway]      我们的入口：认证、限流、请求 id
[Payment Service]  核心：校验、幂等、状态机、编排。不碰卡号
[Risk Service]     风控：给每笔支付打分，决定放行、拒绝或要求额外验证
[Card Vault]       卡号保险库：加密存卡号，发 token，唯一存卡号的地方
[Bank Adapter]     调银行的适配层：向 vault 换卡号，调银行，翻译结果
[Bank]             银行 / 卡组织
[Payment DB]       支付库：支付、幂等记录、outbox（只有 token、last4、brand）
[Webhook Worker]   异步通知商户
```

### 2.2 整体框架图（先看这张，再看下面的编号说明）

**怎么读：** 方框是组件，箭头是调用方向，括号里的数字是步骤编号。**主流程按 (1)(2)(3)... 的顺序看**；(14)(15) 是异步的后台流程。

```text
  ┌─────────────┐   (1) card number + publishable key               ┌─────────────────────────┐
  │   Browser   │──────────────────────────────────────────────────▶│       Card Vault        │
  │  (iframe)   │◀──────────────────────────────────────────────────│      encrypt + KMS      │
  └─────────────┘   (2) one-time token                              │       + vault DB        │
         │                                             ┌───────────▶│                         │
         │                                             │            └─────────────────────────┘
         │ token handed to merchant page               │                       ▲
         │                                             │ (6) metadata, no PAN  │
         │                                             │                       │
         ▼                                             │                       │
  ┌─────────────┐ (3)   ┌─────────────┐ (4)   ┌─────────────────┐              │
  │  Merchant   │──────▶│ API Gateway │──────▶│ Payment Service │              │
  │   Server    │◀──────│             │◀──────│                 │              │
  └─────────────┘ (13)  └─────────────┘       └─────────────────┘              │ (9) detokenize
        ▲                                       │      │      │                │ (mTLS, audited)
        │                 (5) save, (12) update │      │      │(8) charge      │
        │                      ┌────────────────┘      │      └─────────┐      │
        │                      │                       │                │      │
        │                      │                       │ (7) risk check │      │
        │                      │                       │                │      │
        │ (14) webhook         │                       │                │      │
        │                      ▼                       ▼                ▼      │
        │               ┌─────────────┐         ┌─────────────┐     ┌─────────────────────┐ (10)    ┌───────────┐
        │               │ Payment DB  │         │Risk Service │     │    Bank Adapter     │────────▶│   Bank    │
        │               └─────────────┘         └─────────────┘     └─────────────────────┘◀────────└───────────┘
        │                 │   ▲                                           ▲                 (11)
        │                 │   │                                           │
        │   ┌─────────────┘   │                                           │
        │   │ outbox          │                                           │
        │   ▼                 │                                           │
  ┌───────────────┐           │ (15) scan PENDING                         │
  │Webhook Worker │           │                                           │
  └───────────────┘           │                                           │
                              │                                           │
                        ┌───────────────────┐ (15) ask bank with payment_id
                        │  Recovery Worker  │ ────────────────────────────┘
                        └───────────────────┘
```

**三个阶段：**
- **阶段 A：收卡（1）（2）**：卡号只走"浏览器 -> vault"这一条路。
- **阶段 B：创建支付（3）到（8）、（10）到（13）**：商户只传 token。
- **阶段 C：后台（14）（15）**：通知商户、恢复未完成的支付。

### 2.3 每个编号做了什么

| 编号 | 从 -> 到 | 传什么 | 做什么 |
| --- | --- | --- | --- |
| **(1)** | Browser -> Card Vault | 卡号 + 商户的 publishable key | 用户在 vault 提供的 iframe 里输入卡号，**直接发给 vault**。商户页面的 JS 和服务器都读不到卡号。vault 校验 Luhn，用信封加密保存，另存 last4、bin6、brand 和指纹 |
| **(2)** | Card Vault -> Browser | 一次性 token | 返回随机 token（短有效期）。页面把 token 交给商户服务器（图中 Browser 到 Merchant 的竖线） |
| **(3)** | Merchant Server -> API Gateway | `POST /v1/payments`：token、金额、币种；请求头 `Authorization`（API key）、`Idempotency-Key` | 商户发起支付，**请求里只有 token，没有卡号** |
| **(4)** | API Gateway -> Payment Service | 同上，加认证得到的 `merchant_id`、`request_id` | Gateway 用 API key 的哈希查出 `merchant_id`（认证），限流，生成 `request_id`；校验失败的请求在这里或下一步就 422 返回，不再往下走 |
| **(5)** | Payment Service -> Payment DB | 新建一条支付记录 | 先**校验请求**（收集全部错误）；再**幂等**：`INSERT` 一条 `PENDING`，唯一约束 `(merchant_id, idempotency_key)`。插入成功才继续；冲突就读已有记录，返回原结果或 202 |
| **(6)** | Payment Service -> Card Vault | `GET metadata`（内网 mTLS） | 校验 token（存在吗、属于这个商户吗、没过期吗），拿 last4、brand、bin、指纹。**没有卡号**。失败就终止，不调风控和银行 |
| **(7)** | Payment Service -> Risk Service | 脱敏特征：bin6、last4、brand、指纹、金额、IP、设备 | 风控打分。**DECLINE -> 条件更新为 DECLINED(source=RISK)，不调银行，直接到 (13)**；ALLOW -> 继续。结果存进记录（`risk_decision`），重试不重新评分。超时 -> 默认放行并标记 UNSCORED |
| **(8)** | Payment Service -> Bank Adapter | `payment_id`、token、金额、币种 | 请求去银行授权。`payment_id` 同时是发给银行的幂等引用 |
| **(9)** | Bank Adapter -> Card Vault | token（内网 mTLS，仅 Bank Adapter 在白名单） | **detokenize：换回卡号**，写审计日志。第一次使用时把一次性 token 绑定到这个 `payment_id`。卡号只在 adapter 内存里，用完清除 |
| **(10)** | Bank Adapter -> Bank | 卡号、金额、币种、引用 = `payment_id` | 按银行的格式组装请求，带超时调用银行 |
| **(11)** | Bank -> Bank Adapter -> Payment Service | 银行的回复 | adapter 把结果翻译成 **APPROVED / DECLINED / UNKNOWN**。**超时是 UNKNOWN，不是 DECLINED** |
| **(12)** | Payment Service -> Payment DB | 条件更新 + outbox | `UPDATE ... SET status=? WHERE payment_id=? AND status='PENDING'`，**同一个事务里写一条 outbox 事件**。UNKNOWN 则保持 PENDING |
| **(13)** | Payment Service -> API Gateway -> Merchant Server | 响应 | Authorized / Declined 返回 200；Rejected 返回 422；PENDING 返回 202 |
| **(14)** | Webhook Worker -> Merchant | webhook（签名、`event_id`） | **异步**：Worker 读 outbox 里未发送的事件，POST 给商户，失败退避重试。商户用 `event_id` 去重 |
| **(15)** | Recovery Worker -> Payment DB / Bank | 扫描 PENDING；用 `payment_id` 查银行 | **异步**：扫描超过一定时间的 PENDING，用 lease 抢占，通过 Bank Adapter 用同一个 `payment_id` 向银行确认结果，条件更新状态并写 outbox，之后由 (14) 通知商户 |

**按场景走一遍：**

| 场景 | 经过的编号 |
| --- | --- |
| 收卡 | (1) -> (2) |
| 正常支付成功 | (3) -> (4) -> (5) -> (6) -> (7) -> (8) -> (9) -> (10) -> (11) -> (12) -> (13)，之后 (14) |
| 请求格式错误 | (3) -> (4) -> (5) 之前的校验 -> 422，不调任何下游 |
| 风控拒绝 | (3) -> ... -> (7) DECLINE -> (12) -> (13)，不走 (8) 到 (11) |
| 商户重复请求（同 key） | (3) -> (4) -> (5) 发现已有记录 -> 返回原结果，不走后面 |
| 银行超时 | ... -> (10) 超时 -> (11) UNKNOWN -> (12) 保持 PENDING -> (13) 返回 202；之后 (15) 确认 -> (12) -> (14) |
| 调完银行进程崩溃 | 记录是 PENDING -> (15) 恢复 -> (12) -> (14) |

### 2.4 谁能看到什么

| 角色 | 完整卡号 | token | last4 / brand / bin |
| --- | --- | --- | --- |
| Browser（输入时） | 只输入，不保存 | 是 | - |
| Merchant Server | **否** | 是 | 是 |
| API Gateway / Payment Service / Payment DB / 日志 | **否** | 是 | 是 |
| Risk Service | **否** | 是（或不需要） | 是，加卡指纹 |
| Bank Adapter | 仅调银行那一刻，用完即丢 | 是 | 是 |
| Card Vault | 是（加密保存） | 是 | 是 |

**关键决定：卡号从不经过 Payment Service 和 Risk Service。** 好处：只有 vault 和 Bank Adapter 在 PCI 严格范围内。代价：每次调银行多一次 vault 调用。

## 3. Part A：卡数据怎么存、怎么保护

### 3.1 什么是 tokenization

**用一个没有意义的随机字符串代替卡号去流转，真实卡号只存在 vault。**

```text
 存：卡号 -> vault：1) Luhn 校验  2) 生成随机 token（128 位随机数）
                   3) 加密卡号后存库，连同 token、merchant_id、last4、brand
                   4) 返回 token
 用：Bank Adapter 带 token -> vault：1) 检查调用方是不是 Bank Adapter、token 是不是这个商户的
                                   2) 解密，返回卡号，写审计日志
```

| 问题 | 答案 |
| --- | --- |
| token 怎么来的 | **随机生成**，和卡号没有数学关系 |
| 能从 token 反推卡号吗 | **不能**，只能查 vault |
| 和加密有什么区别 | 密文能用密钥解开；token 只是查表的编号，没有密钥能解 |
| token 被偷了怎么办 | 单独没用：绑定商户，detokenize 只有 Bank Adapter 能调，一次性 token 有短有效期 |

**两种 token：**
- **一次性 token**：浏览器输入卡号后得到，短有效期（如 15 分钟）。
- **可重复使用的 token**：商户用 `POST /v1/cards` 换来的（存卡），绑定商户，用于订阅、复购。

### 3.2 Vault API

| 接口 | 谁调用 | 作用 | 认证 |
| --- | --- | --- | --- |
| `POST /v1/tokens` | 浏览器 hosted fields | 提交卡号，返回一次性 token | 商户的 publishable key（只能建 token，不能扣款） |
| `POST /v1/cards` | 商户服务器 | 把一次性 token 换成可重复使用的 card token | 商户 secret key |
| `GET /internal/cards/{token}/metadata` | **Payment Service、Risk Service** | 返回 last4、brand、bin、expiry、商户归属、指纹。**没有卡号** | 内网 mTLS + 服务白名单 |
| `POST /internal/detokenize` | **只有 Bank Adapter** | token 换卡号 | 内网 mTLS + 服务身份白名单，不对外暴露 |
| `DELETE /v1/cards/{id}` | 商户或合规流程 | 删除卡 | 商户 secret key |

要点：
- **metadata 接口是集成的关键**：Payment Service 要校验 token（存在吗、属于这个商户吗、没过期吗）、要拿 last4 展示、要给风控脱敏特征，都靠它，从不需要卡号。
- detokenize 是内部接口，不进 API Gateway，不接受外部流量。
- **一次性 token 绑定到支付**：第一次使用时，vault 用条件更新把 token 绑到这个 `payment_id`（`WHERE bound_payment_id IS NULL`）。同一个支付后续（超时重试、恢复）可以再次还原；**别的支付不能用这个 token**。这样防止 token 被重复使用，又不影响重试。
- tokenize 接口按 IP 和商户限流，防止被拿来"试卡"（card testing）。

### 3.3 数据模型（vault 自己的库，独立账号、独立网络、独立凭证）

`card_vault`

| 字段 | 说明 |
| --- | --- |
| `card_id` (PK) | token 本身，**随机生成**，不能从卡号推算 |
| `merchant_id` | 归属商户，所有查询都带它 |
| `pan_ciphertext` | 卡号加密后的结果（和有效期一起放在加密载荷里） |
| `dek_wrapped` | 数据密钥（DEK），被 KMS 里的主密钥（KEK）加密后的结果 |
| `key_version` | 用的哪一版 KEK，轮换时用 |
| `pan_fingerprint` | `HMAC(密钥, 卡号)`：识别同一张卡、去重、风控速度检查；不是明文，也不是后四位 |
| `last4`, `bin6`, `brand` | **展示和风控用，单独一列，不加密**（低敏感） |
| `bound_payment_id` | 一次性 token 绑定的支付（见 3.2） |
| `status` | ACTIVE / DELETED |
| `created_at`, `expires_at` | |

不存：**CVV 授权后不存**（PCI 不允许）。CVV 只在内存里短暂存在，授权用完即丢。

### 3.4 加密方式（Envelope Encryption）

```text
存：1. 随机生成 DEK
    2. 用 DEK 以 AES-256-GCM 加密（卡号 + 有效期），AAD = card_id + merchant_id
    3. 用 KMS 的 KEK 加密 DEK -> dek_wrapped
    4. 存 pan_ciphertext、dek_wrapped、key_version
取：1. 向 KMS 请求解开 dek_wrapped（KEK 不离开 KMS/HSM）
    2. 用 DEK 解密 pan_ciphertext
    3. 用完立即清除内存
```

为什么这样做：
- **KEK 不落地**：数据库被拖库，没有 KMS 权限也解不开。
- **AAD 绑定 card_id**：防止有人把一条密文复制到另一条记录上。
- **轮换简单**：换 KEK 只需重新加密 DEK，不用重新加密所有卡号。

取舍：每次还原多一次 KMS 调用（延迟、依赖 KMS 可用性）。

### 3.5 怎么只显示后四位（题目第二问的完整答案）

三层保护，缺一不可：

| 层 | 做法 |
| --- | --- |
| **数据层** | 入库时把 `last4`、`bin6`、`brand` 拆成**单独的列**。展示时只读这几列，**永远不解密卡号**。Payment DB 里也只有 `card_token`、`card_last4`、`card_brand` |
| **接口层** | 响应用白名单 DTO，只放 token、last4、brand、expiry，卡号字段根本不在返回对象里 |
| **日志和代码层** | 卡号有专门类型，`toString` 永远脱敏；日志过滤器匹配 13–19 位且通过 Luhn 的数字替换成 `****`；定期扫描日志和备份 |

```java
// 卡号用专门的类型包起来，toString 永远是脱敏的
public final class CardNumber {
    private final char[] value;
    public CardNumber(String pan) { /* Luhn 校验，失败就拒绝 */ this.value = pan.toCharArray(); }
    public String last4() { return new String(value, value.length - 4, 4); }
    @Override public String toString() { return "****" + last4(); }
    // 不提供 getter 返回完整卡号；还原只在 Bank Adapter 调银行的那一行
}
```

为什么后四位可以显示：信息量小，无法还原卡号，PCI 允许显示前 6 位加后 4 位。

> "At tokenization time we store the last four digits, brand and BIN in separate columns. The rest of the system reads only those, so it never needs the full number. Logs go through a masking filter, and the card number has its own type that always prints as masked."

### 3.6 谁能还原卡号，以及审计

- 只有 **Bank Adapter** 能调 detokenize，通过 mTLS 和服务身份白名单。
- 每次 detokenize 记审计日志：谁、哪张卡、为什么（payment id）、什么时候。**日志里不含卡号**。
- 没有人工解密接口。排查问题用 token 和 last4。
- 监控：detokenize 的次数、被拒绝次数、陌生服务的请求，异常就告警。

### 3.7 PCI 范围（一句话）

收卡号的组件才在严格范围内。用 hosted fields + token，商户和我们的 Payment Service、Risk Service、Payment DB、日志都在范围之外，**只有 vault 和 Bank Adapter 要通过严格审计**，范围越小越好。

### 3.8 Vault 的可用性、轮换和删除

- vault 在**关键路径**：服务无状态、多副本、多可用区；数据库主从，按 `card_id` 分片；**不缓存完整卡号**（可缓存 metadata）。
- vault 或 KMS 不可用：**不绕过**。新支付返回可重试错误（503），商户用同一个幂等键重试。
- **轮换**：KEK 定期轮换，新卡用新版本，旧数据后台重新加密 DEK（`key_version` 标记进度）。
- **删除**：商户要求删卡或卡过期，删密文并作废 token；保留合规要求的交易元数据（token、last4、金额），不含卡号。

## 4. Part B：支付流程和状态正确性

### 4.1 API

```text
POST /v1/payments        Idempotency-Key: <商户生成的唯一值>
{ "card_token": "tok_...", "amount": 1050, "currency": "GBP", "reference": "order-123" }
```

响应：

| 结果 | HTTP | body |
| --- | --- | --- |
| Authorized | 200 | `payment_id, status, last4, amount, currency` |
| Declined（银行或风控拒绝） | 200 | `payment_id, status, decline_code, decline_source (BANK/RISK)` |
| Rejected（请求无效，没调银行） | 422 | `errors: [{field, code, message}]`，一次返回全部 |
| Pending（银行超时，结果未知） | 202 | `payment_id, status: PENDING`，商户用同一个 key 重试或等 webhook |
| 同 key 不同请求体 | 409 | |
| 同 key 的请求还在处理中 | 202 或 409（返回已有 payment_id） | |

金额用 `long` 最小货币单位，不用 double。`GET /v1/payments/{id}` 查询：查询里带认证得到的 `merchant_id`，别人的支付返回 404。

### 4.2 数据模型（Payment DB）

`payments`

| 字段 | 说明 |
| --- | --- |
| `payment_id` (PK) | `pay_<分片号>_<随机串>`，分片号编在 id 里 |
| `merchant_id` | 认证得到，不从请求体取 |
| `idempotency_key` | 唯一约束 `(merchant_id, idempotency_key)` |
| `request_hash` | 请求体哈希，用来识别"同 key 不同请求" |
| `card_token`, `card_last4`, `card_brand` | 只有这些，没有卡号 |
| `amount`, `currency` | 最小货币单位和币种 |
| `status` | PENDING / AUTHORIZED / DECLINED |
| `decline_source`, `decline_code` | BANK 或 RISK，统一的错误码 |
| `risk_decision`, `risk_score` | ALLOW / DECLINE / UNSCORED，**保存下来，重试不重新评分** |
| `bank_reference`, `auth_code` | `bank_reference` = `payment_id`，发给银行的幂等引用 |
| `lease_owner`, `lease_until` | 恢复 worker 抢占用 |
| `created_at`, `updated_at` | |

`outbox_events`：`event_id, payment_id, type, payload, delivered`，**和状态更新在同一个事务里写入**，保证通知不丢。

### 4.3 状态机

```text
                 ┌── 风控拒绝 ──────────────▶ DECLINED (source=RISK)
 PENDING ────────┼── 银行同意 ──────────────▶ AUTHORIZED
                 ├── 银行拒绝 ──────────────▶ DECLINED (source=BANK)
                 └── 银行超时、结果未知 ──▶ 保持 PENDING，恢复 worker 确认
```

规则：状态**只能单向前进**，终态不再变化。所有更新都是条件更新：`UPDATE ... SET status=? WHERE payment_id=? AND status='PENDING'`，影响行数为 0 说明别人已处理。

### 4.4 创建支付的完整流程

```text
 1. Gateway：认证（API key -> merchant_id）、限流、生成 request_id
 2. 校验请求（格式、金额、币种、token 格式），收集全部错误 -> 有错误返回 422，不调任何下游
 3. 幂等：INSERT payments(PENDING) 带唯一约束 (merchant_id, key)
      - 插入成功 -> 我来处理
      - 冲突 -> 读已有记录：终态则返回原结果；PENDING 则返回 202；请求哈希不同则 409
 4. 向 vault 取 metadata：token 存在吗、属于这个商户吗、没过期吗？写入 last4/brand
      - 失败 -> 终态 DECLINED/REJECTED，原因 INVALID_TOKEN（没调银行）
 5. 调 Risk Service（短超时，见第 5 章）-> 保存 risk_decision
      - DECLINE -> 条件更新为 DECLINED(RISK)，返回，不调银行
 6. 调 Bank Adapter（带超时）：payment_id 作为银行引用
      - APPROVED/DECLINED -> 条件更新状态，同一事务写 outbox
      - UNKNOWN（超时）-> 保持 PENDING，返回 202
 7. 返回响应；Webhook Worker 异步通知商户
```

**为什么先落库再做其他事：** 任何时候数据库里都有这次操作的记录；崩溃后恢复 worker 才知道有哪些未完成的事。

**事务要短：** 步骤 3 的插入一个事务，步骤 6 的更新另一个事务，**不要在事务里等 vault、风控或银行**，否则连接池被占满。

### 4.5 重试的三层

| 谁重试 | 问题 | 做法 |
| --- | --- | --- |
| **商户 -> 我们** | 同一次付款被请求两次 | `Idempotency-Key`，唯一约束 `(merchant_id, key)`，**按商户隔离**；同 key 同请求返回原结果；同 key 不同请求（比较哈希）返回 409 |
| **我们 -> 银行** | 超时，不知道银行有没有扣 | 先落库 PENDING，`payment_id` 作银行引用；**超时不等于失败**，状态保持 PENDING，不重新扣 |
| **崩溃或丢结果** | 调完银行进程挂了 | 恢复 worker 扫描 PENDING，抢占后用 `payment_id` 向银行查询（或重发，由银行幂等返回原结果），确认后条件更新状态 |

**并发相同 key：** 靠唯一约束原子抢占，先插入成功的处理，另一个读已有记录。**不能"先查再插"**，两个请求会同时查不到然后都插入。

**恢复 worker：**
- 扫描超过一定时间的 PENDING（比如 30 秒以上）。
- 用 lease 抢占（`UPDATE ... SET lease_owner=?, lease_until=now()+30s WHERE status='PENDING' AND lease_until < now()`），多个 worker 不会处理同一条。
- 用 `payment_id` 向银行查询；确认后更新状态并写 outbox。
- 太久还是 PENDING：告警，人工介入；银行没有查询接口时靠对账兜底。

**重试路径里不需要卡号：** 记录里只有 token，恢复时 Bank Adapter 再向 vault 还原。

**对账兜底：** 每天用银行报告和我们的记录比对，发现"银行扣了、我们没记"的差异。

### 4.6 Bank Adapter：它做什么

**Payment Service 只说"对这个 token 扣多少钱"，Bank Adapter 负责变成银行能懂的请求，发出去，把回复翻译回来。**

1. **隔离卡号**：只有它需要卡号。
2. **隔离银行差异**：协议、字段、错误码因银行而异，都收在 adapter 里；加一家银行只加一个实现。

| 规则 | 原因 |
| --- | --- |
| Adapter 无状态，不写支付状态 | 状态只由 Payment Service 条件更新，避免两处写 |
| 发银行时带 `payment_id` 作幂等引用 | 重试或恢复时银行能识别同一笔 |
| 超时、连接断开返回 **UNKNOWN，不是 FAILED** | 不知道银行有没有扣 |
| 银行明确拒绝映射成统一的 decline code | 商户只看到一套错误码 |
| 每家银行单独的超时、连接池、熔断、限流 | 一家慢，不拖垮其他银行 |
| 卡号只放局部变量，用完清零，不写日志、不缓存 | 缩小泄露面 |

> [TODO] 真实银行的协议（ISO 8583、HTTP API）和是否支持幂等引用，要以实际银行为准。面试时说 "depends on the bank; if it has no idempotency support, I'd query by our reference before any retry"。

## 5. Risk Service 集成（之前没特别练的部分）

### 5.1 风控放在哪里，为什么

**位置：在落库 PENDING 之后，调银行之前。**

```text
 校验 -> 幂等 -> vault metadata -> 落库 PENDING -> 【风控】 -> 调银行
```

为什么：
- 落库之后：风控结果可以保存在这条记录上，重试时直接用，不重新评分，保证同一个 key 的结果一致。
- 调银行之前：高风险就不用调银行，省钱，也避免产生不必要的授权。
- 在 vault metadata 之后：风控需要卡的脱敏特征。

### 5.2 给风控什么数据（原则：不给卡号）

| 类别 | 内容 |
| --- | --- |
| 卡特征 | `bin6`（发卡国家、类型）、`last4`、`brand`、**`pan_fingerprint`**、有效期 |
| 交易 | 金额、币种、商户 id、时间、`payment_id` |
| 环境 | 客户 IP、设备指纹、邮箱哈希（由商户或 hosted fields 提供） |
| 历史 | 同一指纹在最近 N 分钟的次数、失败次数、跨商户的使用情况 |

**为什么用 `pan_fingerprint`：** 风控最常见的规则是"同一张卡在短时间内被尝试很多次"（速度检查）。有了指纹，不需要卡号也能认出"同一张卡"。这样风控服务**不进入 PCI 范围**。

### 5.3 风控返回什么，我们怎么处理

| 风控结果 | 我们的动作 |
| --- | --- |
| **ALLOW** | 继续调银行 |
| **DECLINE** | 条件更新为 DECLINED（`decline_source=RISK`），返回 200 Declined，**不调银行** |
| **REVIEW / 需要 3DS**（范围外，但要提到） | 需要增加一个"需要额外验证"的状态，商户完成验证后再继续；先说明位置，不展开 |
| 超时或不可用 | 见 5.4 |

风控拒绝是**明确的终态**，不是"未知"。必须**保存 `risk_decision` 和 `risk_score`**：
- 重试同一个 key 时返回同样结果，不再次评分（评分可能因时间变化而变，会导致同一 key 不同结果）。
- 事后排查、商户争议时有依据。

### 5.4 风控挂了或很慢怎么办（降级策略，必问）

| 选择 | 做法 | 取舍 |
| --- | --- | --- |
| **默认放行（fail open）** | 超时（如 200ms 内没回）就继续，标记 `risk_decision=UNSCORED`，告警 | 不影响支付成功率；银行自己也有风控，有一定兜底；风险是欺诈暂时漏过 |
| 默认拒绝（fail closed） | 风控不可用就拒绝 | 更安全，但风控一挂支付全停，损失大 |
| **推荐** | 默认放行，**加本地简单规则兜底**（金额上限、黑名单、每商户或每卡速率限制），**高风险商户可配置为 fail closed** | 平衡可用性和风险 |

说法：`"Correctness for money comes first, but a risk outage shouldn't stop all payments. So I fail open with local fallback rules, mark the payment unscored, and alert. High-risk merchants can be configured to fail closed."`

### 5.5 技术细节

- **同步调用，短超时**（百毫秒级），因为在支付关键路径上。
- 风控服务要**幂等**：用 `payment_id` 作引用，同一支付重复评分得到相同结果或明确返回已有结果。
- 对风控服务加**熔断和独立连接池**（和银行一样，别让它拖垮 Payment Service）。
- **反馈闭环（异步）**：支付最终结果（AUTHORIZED、DECLINED、后续 chargeback）通过事件流（outbox -> Kafka）发给风控，供模型学习和规则更新。这条是异步的，不在关键路径。
- **vault 的 tokenize 接口也要风控**：攻击者用大量卡号去试（card testing）。对 tokenize 按 IP、商户限流，指纹的失败率异常时拦截。

### 5.6 风控与"状态正确"的关系

| 问题 | 答案 |
| --- | --- |
| 风控结果会影响重试吗？ | 会，所以保存并复用，重试不重新评分 |
| 风控拒绝后商户重试同一个 key？ | 返回原来的 Declined(RISK)，不再调风控或银行 |
| 风控通过但银行超时？ | 保持 PENDING，恢复 worker 确认；风控决定已保存，不重评 |
| 崩溃发生在风控之后、调银行之前？ | 记录是 PENDING 且已有 `risk_decision=ALLOW`，恢复 worker 直接去调银行（用 `payment_id` 查询确认银行还没处理） |

## 6. 横切：认证、授权、扩展性、可观测、测试

### 6.1 认证和授权

- **商户认证：** API key 由平台生成，库里只存 SHA-256 哈希（key 是高熵随机数，不需要 bcrypt）；请求到来时对 key 算哈希、查 `api_keys` 得到 `merchant_id`，放进请求上下文。`merchant_id` **只来自认证结果，不信请求体**。
- **授权（防 IDOR）：** 每个按 id 的查询都带 `AND merchant_id = 当前商户`，别人的返回 404（不是 403）。token 和幂等键都绑定 `merchant_id`。
- **服务间：** 内部调用用 mTLS 和服务白名单；detokenize 只允许 Bank Adapter，metadata 允许 Payment Service 和 Risk Service。
- 详见 `payment-endpoint-validation.md` §11。

### 6.2 扩展性（先找瓶颈）

| 位置 | 做法 |
| --- | --- |
| Gateway、Payment Service、Bank Adapter、Risk 调用层 | **无状态**，加机器 |
| Payment DB | 索引和短事务 -> 读写分离（写后读走主库）-> **按 transaction id 分片**（分片号在 id 里；创建时按 `hash(merchant_id, key)` 路由，保证同 key 重试落同一分片）；按商户的报表走 CDC 同步的读库 |
| Vault | 无状态多副本，数据库按 `card_id` 分片，不缓存卡号 |
| 银行、风控（外部或慢） | 超时、**每个下游单独连接池（舱壁）**、熔断、退避重试、按商户限流 |
| 非关键路径 | webhook、对账、风控反馈走 outbox 加队列，和主流程解耦 |

详见 `payment-endpoint-validation.md` §12 和 `payment-processor-design-prep.md` §6。

### 6.3 可观测

- 日志：JSON，带 `request_id`、`payment_id`、`idempotency_key`、`merchant_id`，**不含卡号和 CVV**。
- 指标：支付成功率、各阶段延迟（vault、风控、银行）、PENDING 数量和**最老年龄**、风控超时率、`UNSCORED` 占比、detokenize 的拒绝数。
- 告警：PENDING 积压、风控或银行错误率上升、陌生服务调用 detokenize、日志里发现疑似卡号。

### 6.4 测试

| 类别 | 用例 |
| --- | --- |
| 校验 | 多个错误一次返回；无效请求时银行和风控调用次数为 0 |
| 幂等 | 同 key 重复请求银行只被调一次；同 key 不同请求体返回 409；并发相同 key 只有一个处理 |
| 风控 | 拒绝时不调银行；超时放行并标记 UNSCORED；重试不重新评分 |
| 银行 | 同意、拒绝、超时保持 PENDING（不是 Declined） |
| 恢复 | 模拟崩溃在银行调用后，恢复 worker 更新状态；两个 worker 不重复处理 |
| 安全 | 响应和日志里没有完整卡号；商户 A 读不到商户 B 的支付 |

## 7. 失败场景总表

| 场景 | 处理 |
| --- | --- |
| 商户重复请求 | 幂等键，返回原结果 |
| 两个相同 key 并发 | 唯一约束抢占，另一个读已有记录 |
| vault 不可用 | 返回 503，可重试；不绕过 |
| 风控超时 | fail open + 本地规则 + UNSCORED + 告警；高风险商户 fail closed |
| 银行超时 | 保持 PENDING，不是 Declined；恢复 worker 确认 |
| 调完银行进程崩溃 | PENDING 记录 + 恢复 worker + 银行引用查询；对账兜底 |
| 数据库被拖库 | Payment DB 里没有卡号；vault 里只有密文，没有 KMS 权限解不开 |
| Bank Adapter 被入侵 | 白名单、限速、审计、吊销身份 |
| token 被商户 B 盗用 | `merchant_id` 不一致就拒绝；一次性 token 绑定 payment |
| 日志漏出卡号 | Luhn 扫描告警，清理日志 |
| 客户端 webhook 失败 | outbox 重试、签名、`event_id` 去重 |

## 8. 5 分钟英文口述稿

> **Clarify.** Who sends the card data, the merchant's server or the browser? I'll assume the browser, through hosted fields, so the merchant never sees it. I'll also assume a few hundred TPS with 1–2k at peak, a ready risk service, and I'll leave 3DS and settlement out. Does that match what you had in mind?
>
> **Architecture.** There are five main parts: the API gateway, the payment service, a risk service, a bank adapter, and a card vault. The key decision is that the full card number lives only in the vault. Everyone else handles a token.
>
> **Card storage.** The vault is isolated, with its own network, database and credentials. It uses envelope encryption: a data key encrypts the card, and a master key in KMS encrypts the data key. It returns a random token tied to the merchant. Only the bank adapter can detokenize, over mutual TLS, and every call is audited.
>
> **Last four digits.** When we tokenize, we store the last four digits, the BIN and the brand in separate columns. The payment service reads those through a metadata endpoint, so it never needs the full number. Logs go through a masking filter, and the card number has its own type that always prints as masked.
>
> **Payment flow and retries.** The merchant sends the token, the amount and an idempotency key. We insert the payment as pending, with a unique constraint on merchant id and key, so concurrent duplicates are handled by the database. Then we call risk, then the bank adapter. The bank adapter gets the card number from the vault, calls the bank with our payment id as the reference, and returns approved, declined or unknown. A timeout is unknown, not failed. We keep it pending, and a recovery worker with a lease asks the bank using the same reference. Status updates are conditional and only move forward, and reconciliation is the last safety net.
>
> **Risk.** The risk check runs after we save the payment and before we call the bank. We send only masked features: BIN, last four, brand and a keyed fingerprint of the card, plus amount, IP and device, so risk is outside PCI scope. We save the decision, so a retry doesn't re-score. If the risk service times out, we fail open with local rules, mark the payment unscored, and alert. High-risk merchants can fail closed.
>
> **Scaling.** The services are stateless, so I add instances. The database is the first bottleneck, so I'd add indexes and short transactions, then replicas, then shard by transaction id. Each external dependency gets its own timeout, connection pool and circuit breaker.
>
> **Trade-offs.** Every payment depends on the vault, so it must be highly available, and I never cache the card number. Does this cover what you wanted, or should I go deeper on one part?

## 9. 面试官可能追问

| 追问 | 答案要点 |
| --- | --- |
| 为什么不直接加密后存在业务库？ | 业务库访问面大，会把整个系统拉进 PCI 范围；隔离后只有 vault 要审计 |
| 后四位为什么不每次解密取？ | 展示是高频读，每次解密扩大风险面，增加 KMS 调用；后四位低敏感，单独存 |
| token 为什么要随机？ | 由卡号推算（哈希或可逆加密）的 token，拿到后可能反推卡号 |
| 怎么识别同一张卡？ | 带密钥的 HMAC 指纹，不用明文，也不用后四位（会重复） |
| 风控为什么不给卡号？ | 风控不应进入 PCI 范围；指纹、BIN、last4 足够做速度和地区规则 |
| 风控挂了怎么办？ | 默认放行 + 本地规则 + 标记未评分 + 告警；高风险商户可 fail closed |
| 风控的结果重试时怎么办？ | 保存在记录上，重试直接复用，不重新评分 |
| 一次性 token 重试会不会失效？ | 第一次使用时绑定 payment_id，同一支付可重复还原，别的支付不能用 |
| 银行超时为什么不返回 Declined？ | 不知道有没有扣款，返回 Declined 商户可能换 key 重试，造成重复扣款 |
| 两个并发相同 key 怎么办？ | 唯一约束原子抢占，不是先查再插 |
| 崩溃在银行调用之后怎么办？ | PENDING 记录加恢复 worker 加银行引用查询，对账兜底 |
| 为什么按 transaction id 分片？ | 一笔支付和它的 capture、refund 在一起，大商户被打散；商户报表走读库 |
| 创建时没有 id，怎么保证幂等？ | 按 `hash(merchant_id, key)` 路由到分片，同 key 重试落同一分片 |
| vault 成为单点怎么办？ | 无状态多副本、多可用区、数据库主从；失败时明确返回，商户用同一个 key 重试 |
| 密钥泄露怎么办？ | 轮换 KEK，重新加密 DEK；分层密钥缩小影响；检测和告警 |
| 内部人员怎么防？ | 没有人工解密接口，最小权限，全量审计，敏感操作双人审批 |
| 退款或 capture 还需要卡号吗？ | 一般用银行返回的引用，不需要卡号；需要时用 token 通过 Bank Adapter 换 |
| 如果商户不用 hosted fields，直接传卡号？ | 只对合格商户开放，在入口立刻转给 vault，关掉该接口的请求体日志；双方 PCI 范围变大 |

## 10. 待确认（不要编）

- [TODO] 你们实际的 vault、风控和 Bank Adapter 的做法，和上面是否一致。
- [TODO] 面试时的真实银行协议和风控接口，以面试官提供的为准。
- [TODO] "Bank Adapter 向 vault 换卡号"是本设计的假设，面试官可能有别的假设（例如银行直接接受 token），遇到时按对方的来。
