# Design: Payment Gateway for Thousands of Merchants (Card Vault focus)

> Practice question: Design the gateway for thousands of merchants: where card data is stored, how you keep only the last four digits visible, and how a payment's status stays correct across retries.

说明：这是通用的行业做法，不是我们公司的实际设计。面试时说 "I'd design it this way"，不要说成"我们做过"。完整的支付流程、幂等、数据模型见 `payment-processor-design-prep.md`，这里只深挖 **Card Vault**，并把三个问题串成一条线。

## 0. 三个问题，一条主线

```text
卡号只在 vault 里 -> 其他系统只拿 token + last4 -> 重试时只带 token，状态靠幂等键和状态机
```

- Q1 卡号存哪里 -> **隔离的 Card Vault**，加密，只有它能还原卡号。
- Q2 只显示后四位 -> **入库时就把展示字段拆出来单独存**，其他系统从不接触完整卡号。
- Q3 重试时状态正确 -> **幂等键 + 先落库的 operation + 条件更新 + 恢复 worker + 对账**，重试路径里不需要卡号。

## 1. Clarify（先问）

1. 卡号由谁收集？商户服务器，还是浏览器直接送给我们（hosted fields）？
2. 规模：几千个商户，峰值 TPS 大概多少？（先假设，白板上写明。）
3. 愿意承担多大的 PCI 范围？目标是越小越好。
4. 要不要支持存卡（以后再扣）？
5. 范围外：3DS、风控、对账先不展开。

> "Before I start, can I confirm who sends the card data: the merchant's server or the browser? And I'd like to keep our PCI scope as small as possible."

## 2. 总体架构

**怎么读：** 先看 2.1 的"有哪些角色"，再按 2.2 的三个步骤从上往下看。每一步只画一条流程，箭头上写的是**传了什么**。

### 2.1 角色（先认人）

```text
[Browser]          用户的浏览器，页面里嵌着 vault 提供的输入框（hosted fields）
[Merchant Server]  商户自己的后端
[API Gateway]      我们的入口：认证、限流
[Payment Service]  我们的支付核心：幂等、状态机、写 DB（**不碰卡号**）
[Bank Adapter]     调银行的适配层
[Card Vault]       卡号保险库：加密存卡号，发 token（**唯一存卡号的地方**）
[Bank]             银行 / 卡组织
```

### 2.2 三步流程

**步骤 1：收卡。卡号只走浏览器到 vault 这一条路**

```text
 [Browser]
    │  (1) 用户输入卡号。卡号 ──────────────▶ [Card Vault]   加密保存卡号
    │                                              │
    │  (2) 返回 token  ◀───────────────────────────┘
    ▼
 [Browser] 拿到 token，把 token 交给商户页面

 商户服务器全程看不到卡号。
```

**步骤 2：发起支付。只传 token，不传卡号**

```text
 [Merchant Server]
    │  (3) token + 金额 + 幂等键
    ▼
 [API Gateway]            认证、限流
    │
    ▼
 [Payment Service]        幂等检查 -> 写 DB（只存 token、last4、brand、状态）
    │
    ▼
 [Bank Adapter]           准备调银行
```

**步骤 3：调银行。卡号只在这一步短暂出现**

```text
 [Bank Adapter]
    │  (4) token ───────────────▶ [Card Vault]   检查权限 + 写审计日志
    │                                   │
    │  (5) 卡号 ◀───────────────────────┘        (内部 mTLS，只有 Bank Adapter 能调)
    │
    │  (6) 卡号 ─────────────────▶ [Bank]
    │
    │  (7) 用完立刻从内存清除卡号
    ▼
 [Bank Adapter] 把结果交给 Payment Service 更新状态
```

### 2.3 一眼看懂：谁能看到什么

| 角色 | 完整卡号 | token | last4 / brand |
| --- | --- | --- | --- |
| Browser（输入时） | 是（只输入，不保存） | 是 | - |
| Merchant Server | **否** | 是 | 是 |
| API Gateway / Payment Service / DB / 日志 | **否** | 是 | 是 |
| Bank Adapter | 仅调银行那一刻，用完即丢 | 是 | 是 |
| Card Vault | 是（加密保存） | 是 | 是 |

| 组件 | 职责 | 在 PCI 范围内？ |
| --- | --- | --- |
| Card Vault | 收卡号、加密存储、发 token、受控还原 | **是** |
| Bank Adapter | 调银行前向 vault 换卡号，用完即丢 | 是（范围尽量小） |
| API Gateway、Payment Service、DB、日志 | 只见 token、last4、brand | **否** |

关键决定：**卡号从不经过 Payment Service。** 好处：只有两个小组件要通过 PCI 审计。代价：每次调银行多一次 vault 调用。

## 3. Card Vault 详细设计

### 3.1 API

| 接口 | 谁调用 | 作用 | 认证 |
| --- | --- | --- | --- |
| `POST /v1/tokens` | 浏览器 hosted fields | 提交卡号，返回一次性 token（短有效期，如 15 分钟） | 商户的 publishable key（只能建 token，不能扣款） |
| `POST /v1/cards` | 商户服务器（存卡） | 把一次性 token 换成可重复使用的 card token，绑定商户 | 商户 secret key |
| `POST /internal/detokenize` | **只有 Bank Adapter** | token 换成卡信息 | 内网 mTLS + 服务身份白名单，不对外暴露 |
| `DELETE /v1/cards/{id}` | 商户或合规流程 | 删除卡 | 商户 secret key |

要点：
- 一次性 token 防止被拿去反复使用；存卡才生成可重复使用的 token。
- token **绑定商户**：商户 A 的 token 给商户 B 用，直接拒绝（授权检查，防 IDOR）。
- detokenize 是内部接口，不进 API Gateway，也不接受外部流量。

### 3.2 数据模型（vault 自己的库，独立账号、独立网络、独立凭证）

`card_vault`

| 字段 | 说明 |
| --- | --- |
| `card_id` (PK) | token 本身。**随机生成**（如 128 位随机数），不能从卡号推算 |
| `merchant_id` | 归属商户，所有查询都带它 |
| `pan_ciphertext` | 卡号加密后的结果（和有效期一起放在加密载荷里） |
| `dek_wrapped` | 数据密钥（DEK），被 KMS 里的主密钥（KEK）加密后的结果 |
| `key_version` | 用的是哪一版 KEK，轮换时用 |
| `pan_fingerprint` | `HMAC(密钥, 卡号)`，用来识别同一张卡、去重；不是明文，也不是后四位 |
| `last4`, `bin6`, `brand` | **展示用，单独一列，不加密**（低敏感） |
| `status` | ACTIVE / DELETED |
| `created_at`, `updated_at` | |

不存：**CVV 授权后不存**（PCI 不允许）。一次性 token 的 CVV 只在内存里短暂存在，授权用完即丢。

### 3.3 加密方式（Envelope Encryption）

```text
存：
  1. 随机生成 DEK
  2. 用 DEK 以 AES-256-GCM 加密 (卡号 + 有效期)，AAD = card_id + merchant_id
  3. 用 KMS 的 KEK 加密 DEK -> dek_wrapped
  4. 存 pan_ciphertext、dek_wrapped、key_version
取：
  1. 向 KMS 请求解开 dek_wrapped（KEK 不离开 KMS/HSM）
  2. 用 DEK 解密 pan_ciphertext
  3. 用完立即清除内存
```

为什么这样做：
- **KEK 不落地**：数据库被拖库，没有 KMS 权限也解不开。
- **AAD 绑定 card_id**：防止有人把一条密文复制到另一条记录上。
- **轮换简单**：换 KEK 时，只需要重新加密 DEK，不用重新加密所有卡号。

取舍：每次还原多一次 KMS 调用（延迟、依赖 KMS 可用性）。

### 3.4 怎么只显示后四位（Q2 的完整答案）

三层保护，缺一不可：

| 层 | 做法 |
| --- | --- |
| **数据层** | 入库时就把 `last4`、`bin6`、`brand` 拆成**单独的列**。展示时只读这几列，**永远不解密卡号**。Payment Service 的表里也只有 `card_token`、`card_last4`、`card_brand` |
| **接口层** | 响应用白名单 DTO，只放 token、last4、brand、expiry 等，卡号字段根本不在返回对象里 |
| **日志和代码层** | 见下 |

代码层的两个实用做法（AI 辅助编码面试里很好讲）：

```java
// 卡号用专门的类型包起来，toString 永远是脱敏的，也不可序列化
public final class CardNumber {
    private final char[] value;
    public CardNumber(String pan) { /* Luhn 校验，失败就拒绝 */ this.value = pan.toCharArray(); }
    public String last4() { return new String(value, value.length - 4, 4); }
    @Override public String toString() { return "****" + last4(); }
    // 不提供 getter 返回完整卡号给日志；还原只在 Bank Adapter 调银行的那一行
}
```

- 日志框架加**过滤器**：匹配 13–19 位数字且通过 Luhn 校验的内容，直接替换成 `****`。
- 请求体的卡号字段在记录前清掉。
- 定期**扫描日志和数据库备份**，发现疑似卡号就告警。
- 为什么后四位可以显示：后四位信息量很小，无法还原卡号，PCI 允许显示（前 6 位加后 4 位）。

> "At tokenization time we store the last four digits, brand and BIN in separate columns. The rest of the system only reads those columns, so it never needs the full number. Logs go through a masking filter, and the card number has its own type that always prints as masked."

### 3.5 谁能还原卡号，以及审计

- 只有 **Bank Adapter** 能调 detokenize，通过 mTLS 和服务身份白名单。
- 每次 detokenize 记审计日志：谁、哪张卡、为什么（payment id）、什么时候。**日志里不含卡号**。
- 没有人工解密接口。排查问题用 token 和 last4。
- 监控：detokenize 的次数、被拒绝的次数、来自陌生服务的请求，异常就告警。

### 3.6 可用性与规模

- 每次付款都要调 vault，所以它是**关键路径**。
- vault 服务**无状态**，多副本，多可用区；数据库主从，按 `card_id` 分片。
- **不缓存完整卡号**。可以缓存元数据（last4、brand）。
- 容量估算（先假设，白板上写明）：detokenize 次数约等于调银行的次数，和支付 TPS 同量级；存卡的数据量按"每张卡几百字节"估算，增长很慢。
- vault 或 KMS 不可用：**不绕过**。新支付返回可重试的错误（503），商户带着同一个幂等键重试即可。

### 3.7 密钥轮换与删除

- **轮换**：KEK 定期轮换，新的卡号用新版本，旧数据后台重新加密 DEK（`key_version` 标记进度）；新旧版本短时间并存。
- **删除**：商户要求删卡，或卡过期，删除密文并作废 token；这相当于销毁数据（密钥和密文都没了）。保留合规要求的交易元数据（token、last4、金额），不含卡号。

### 3.8 失败场景

| 场景 | 处理 |
| --- | --- |
| 数据库被拖库 | 只有密文，没有 KMS 权限无法解密；告警，轮换密钥 |
| 某个 Bank Adapter 被入侵 | 白名单、限速、审计，可以快速吊销它的身份 |
| KMS 慢或挂 | vault 返回可重试错误；多区域部署 KMS |
| token 被商户 B 盗用 | 授权检查：`merchant_id` 不一致就拒绝 |
| 日志里漏出卡号 | Luhn 扫描告警，修复并清理日志 |

## 4. 重试时状态保持正确（Q3 的完整答案）

重试有三层，每层规则不同：

| 谁重试 | 问题 | 做法 |
| --- | --- | --- |
| **商户 -> 我们** | 同一次付款被请求两次 | 必须带 `Idempotency-Key`，唯一约束 `(merchant_id, key)`，**按商户隔离**；同 key 同请求返回原结果，同 key 不同请求（比较请求哈希）返回 409 |
| **我们 -> 银行** | 超时，不知道银行有没有扣 | 先落库（operation = PENDING，带银行幂等引用），再调银行；**超时不等于失败**，状态保持 PENDING，不重新扣 |
| **崩溃或丢结果** | 调完银行进程挂了 | 恢复 worker 扫描 PENDING，用 lease 抢占，用银行幂等引用或查询接口确认结果，再更新状态 |

状态正确性的保证：
1. **先落库再调银行**：任何时候数据库里都有这次操作的记录。
2. **条件更新**：`UPDATE ... SET status='CAPTURED' WHERE id=? AND status='PENDING'`，影响行数为 0 说明别人已处理，防止重复更新和状态倒退。
3. **状态机单向前进**：PENDING -> 成功或失败，不能回头。
4. **重试路径里不需要卡号**：operation 里存的是 token，恢复时 Bank Adapter 再向 vault 换一次。
5. **对账兜底**：每天用银行报告和我们的记录比对，发现"银行扣了、我们没记"的差异。

> "Retries happen at three levels. Merchants retry with an idempotency key. We write the operation as PENDING before calling the bank, so a timeout is not treated as a failure. And a recovery worker with a lease resolves any pending work by asking the bank with the same reference. Reconciliation catches anything left."

详细流程和状态机见 `payment-processor-design-prep.md` 第 4、5 章。

## 5. 横切检查（skill 的清单）

| 维度 | 本设计里的体现 |
| --- | --- |
| 认证和授权 | 商户 API key 认证；token 和幂等键都绑定 `merchant_id`；detokenize 只允许 Bank Adapter |
| 金额 | 整数最小货币单位加币种，不用 double |
| 限流 | 按商户限流（防刷）；vault 对 tokenize 按 IP 和商户限流，防止被用来"试卡" |
| 超时与降级 | 调 vault、银行都有超时和断路器；vault 不可用就明确失败，不绕过 |
| 可测试 | Vault、KMS、Bank 都在接口后面，可以 mock；测试"日志里不出现卡号"；幂等重复请求测试 |
| 可观测 | 指标：支付成功率、detokenize 延迟和错误率、PENDING 数量和年龄；告警：PENDING 积压、拒绝异常增加；日志带 request id、不含卡号 |
| 可扩展 | 服务无状态；按 `card_id` 和 `processor_transaction_id` 分片；热点是 vault 和银行 |

## 6. 3–4 分钟口述稿

> **Clarify.** First, who sends the card data, the merchant's server or the browser? I'd have the browser send it directly to us through hosted fields, so the merchant's server never sees it and our PCI scope stays small.
>
> **Card storage.** I'd isolate the card data in a separate vault with its own network, database and credentials. It encrypts each card number with envelope encryption: a data key encrypts the number, and a master key in KMS encrypts the data key. The vault returns a random token that is tied to the merchant. Everything else, like the payment service, only sees the token.
>
> **Last four digits.** At tokenization time the vault also stores the last four digits, BIN and brand as separate columns. The rest of the system reads those, so it never needs the full number. The card number has its own type that always prints as masked, and logs go through a masking filter that detects card numbers with a Luhn check.
>
> **Who can decrypt.** Only the bank adapter, over an internal authenticated call, and every call is audited. If the vault is down, new payments fail with a retryable error. We never bypass it.
>
> **Retries.** Merchants must send an idempotency key, unique per merchant. We write the operation as pending before calling the bank, so a timeout is not a failure. A recovery worker resolves pending operations using the bank's reference, and conditional updates keep the status moving only forward. Reconciliation is the last safety net.
>
> **Trade-off.** Every payment depends on the vault, so it has to be highly available, and I never cache the full number. Does this match what you had in mind?

## 7. 面试官可能追问

| 追问 | 答案要点 |
| --- | --- |
| 为什么不直接加密后存在业务库？ | 业务库的访问面太大（很多服务和人），会把整个系统拉进 PCI 范围；隔离后只有 vault 要审计 |
| 后四位为什么不每次解密取？ | 展示是高频读，每次解密会扩大风险面，增加 KMS 调用；后四位低敏感，单独存就够 |
| token 为什么要随机？ | 如果由卡号推算（哈希、可逆加密），拿到 token 就可能反推卡号 |
| 怎么识别同一张卡？ | 带密钥的 HMAC 指纹，不用明文，也不用后四位（会重复） |
| 密钥泄露怎么办？ | 轮换 KEK，重新加密 DEK；分层密钥缩小影响；检测和告警 |
| 能每个商户一把密钥吗？ | 可以，隔离更好；代价是密钥数量和管理复杂度增加 |
| 内部人员怎么防？ | 没有人工解密接口，最小权限，全量审计，敏感操作双人审批 |
| vault 成为单点怎么办？ | 无状态多副本、多可用区、数据库主从；失败时明确返回并让商户用同一幂等键重试 |
| 一次性 token 和存卡 token 的区别？ | 一次性：短有效期，防滥用；存卡：绑定商户，可重复使用，可删除 |
| 退款或 capture 时还需要卡号吗？ | 一般用银行返回的引用，不需要卡号；需要时用 token 通过 Bank Adapter 换 |

## 8. 待确认（不要编）

- [TODO] 你们公司的真实 vault 是内部服务还是第三方？如果被问到真实经历，只说你确认的部分。
- [TODO] Bank Adapter 向 vault 换卡号这一点，和你们实际设计是否一致。
- [TODO] 面试官给出的 TPS 和商户规模，白板上以对方给的为准。
