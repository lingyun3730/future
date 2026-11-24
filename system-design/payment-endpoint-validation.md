# Practice: POST payment endpoint, gateway vs bank checks

> Build a POST endpoint that validates a card payment, sends it to the bank simulator and returns Authorized or Declined, or Rejected without calling the bank when the request is invalid.
> Which of these checks belong in the gateway, which belong to the bank, and how do you report several failures in one response?

说明：通用做法，不是你们公司的实际设计。

## 0. 先和面试官沟通：可以问的问题

开头问 3–5 个，每个带"我的默认假设"，对方不回答也能往下走。每个问题要说明"它会影响我的设计"。

开场：`"Before I start, I'd like to clarify a few things so I design the right thing."`

| # | 问题 | English | 默认假设 | 影响什么设计 |
| --- | --- | --- | --- | --- |
| 1 | 请求有多个错误时，要一次全部返回吗？ | If the request has several problems, should I return all of them at once? | 全部返回 | 校验器收集错误，不 fail-first |
| 2 | 银行模拟器超时，应该返回什么？ | What should I return if the bank simulator times out? | 既不是 Declined 也不是 Rejected，返回 Pending/504 | 状态和错误处理 |
| 3 | 哪些检查你希望放在 gateway，哪些留给银行？ | Which checks do you expect in the gateway, e.g. Luhn and expiry, and which should be left to the bank? | Gateway 查格式和范围，银行查余额等 | 校验范围 |
| 4 | 调用方会重试吗？要支持幂等键吗？ | Can the caller retry? Should I support an idempotency key? | 先支持，范围内只做最小版 | 是否要存请求、返回原结果 |
| 5 | 要持久化支付记录，还是内存即可？ | Should I persist payments, or is in-memory fine for now? | 内存即可，说明生产要落库 | 代码范围 |

可选的追问（时间够再问）：
- 金额的单位和币种怎么传？（我假设最小货币单位的整数、ISO 4217）
- 卡号可以直接出现在请求里吗，还是只收 token？（练习里假设请求里有卡号，但绝不记日志、不返回）
- 支持哪些币种和卡品牌？
- Rejected 用 400 还是 422？响应格式有没有要求？
- 要写测试吗？有没有技术栈限制？

说完假设再动手：
`"I'll assume amounts are integers in minor units, currencies are ISO codes, I never log or return the full card number, and I'll leave 3DS and fraud out. Does this match what you had in mind?"`

## 1. 一句话答案

- **Gateway 查"这个请求本身合不合格"**：格式、必填、范围，不需要查任何外部信息，几毫秒就能判断，错了就不调银行（Rejected）。
- **银行查"这笔钱能不能付"**：余额、卡是否被冻结或挂失、风控、CVV 是否匹配、3DS。只有银行有这些信息，结果是 Authorized 或 Declined。
- **多个错误一次返回**：校验时不要遇到第一个错误就停，收集成列表，一次返回，每个错误带 `field`、`code`、`message`。

## 2. 三种结果，别混在一起

| 结果 | 含义 | 是否调了银行 | HTTP 状态 |
| --- | --- | --- | --- |
| **Rejected** | 请求本身无效，我们拒绝处理 | **否** | 422（或 400） |
| **Authorized** | 银行同意 | 是 | 200 |
| **Declined** | 银行拒绝（余额不足等） | 是 | 200 |

要点：
- Declined 是**正常的业务结果**，不是错误，所以用 200。Rejected 是**调用方的错**，用 4xx。
- 银行超时或不可用**既不是 Declined 也不是 Rejected**。不知道银行有没有扣，返回 504 或"处理中"，不要说 Declined（否则商户可能重试，造成重复扣款）。

## 3. 哪些检查放在哪里

### 3.1 Gateway（无状态、便宜、规则稳定）

| 检查 | 规则 | 字段 |
| --- | --- | --- |
| 必填 | 缺字段 | 全部 |
| 卡号格式 | 只有数字，长度 13–19，**Luhn 校验通过** | `card_number` |
| 有效期格式 | 月份 1–12，年份是合法数字 | `expiry_month`, `expiry_year` |
| 有效期未过 | 当月最后一天之前都算有效（注意当月不算过期） | 同上 |
| CVV 格式 | 3 位（Amex 4 位），只有数字 | `cvv` |
| 金额 | 正整数，用最小货币单位（分），不用 double | `amount` |
| 币种 | 是合法的 ISO 4217 且**我们支持** | `currency` |
| 卡品牌 | 我们支持这个品牌 | `card_number` |
| 幂等键 | 请求头必须有 | `Idempotency-Key` |

为什么放这里：
1. **Fail early**：错了就别浪费一次银行调用（有成本、有延迟、可能被银行当作异常交易）。
2. 规则是稳定的行业标准（Luhn、ISO 4217），不依赖银行。
3. 减少银行侧噪音和被限流的风险。

### 3.2 银行（需要账户和风控信息）

| 检查 | 原因 |
| --- | --- |
| 余额或额度是否足够 | 只有银行知道 |
| 卡是否被冻结、挂失、被盗 | 同上 |
| CVV 是否**匹配**这张卡 | 我们只能查格式，不能查对不对 |
| 有效期是否和卡片记录一致 | 我们只能查没过期 |
| 风控或欺诈规则 | 银行自己的模型 |
| 是否需要 3DS | 银行或发卡行决定 |
| 单笔或日限额 | 银行规则 |

**不要在 Gateway 里复制银行规则**：规则会变、会因银行而异，复制了就会和银行不一致，还要不断维护。

### 3.3 灰色地带

| 检查 | 放哪里 | 理由 |
| --- | --- | --- |
| 币种是否支持 | Gateway（我们是否支持）；银行（它是否支持该币种）各自查 | 两个问题：我们愿不愿意收，银行能不能处理 |
| 金额上限 | Gateway 放一个**粗的合理上限**防止输入错误；真正的额度交给银行 | 防手滑，不替银行判断 |
| BIN 是否存在 | 一般交给银行，除非自己维护 BIN 表 | 维护成本高 |

## 4. 多个错误一次返回

**做法：** 校验器收集全部错误，最后统一返回，不要 fail-first。

```json
HTTP 422
{
  "status": "Rejected",
  "errors": [
    { "field": "card_number",  "code": "INVALID_LUHN",     "message": "Card number is not valid" },
    { "field": "expiry_month", "code": "OUT_OF_RANGE",     "message": "Month must be 1-12" },
    { "field": "currency",     "code": "UNSUPPORTED",      "message": "Currency XYZ is not supported" }
  ]
}
```

设计点：
- **稳定的错误码**（`code`）给程序判断，`message` 给人看。
- `field` 指向出错字段，商户可以直接在表单上标红。
- **不要回显完整卡号**或 CVV，错误信息里也不要。
- 多个错误互相依赖时要处理：如果卡号缺失，就不要再报"Luhn 无效"，只报"缺失"。每个字段先查必填，再查格式。
- 有时用 fail-first 更好：安全相关的错误（例如认证失败），不要透露更多信息。

Authorized 与 Declined 的响应：

```json
HTTP 200
{ "status": "Authorized", "payment_id": "pay_...", "authorization_code": "A1B2C3", "last4": "1111", "amount": 1050, "currency": "GBP" }

HTTP 200
{ "status": "Declined", "payment_id": "pay_...", "decline_code": "INSUFFICIENT_FUNDS", "last4": "1111" }
```

- 只返回 `last4`，永远不返回完整卡号。
- `decline_code` 统一成我们自己的一套，不要把银行的原始码直接暴露。

## 5. 处理流程

```text
 POST /payments
   │
   ├─ 1. 认证、读 Idempotency-Key
   ├─ 2. 校验（收集全部错误）──有错误──▶ 422 Rejected（不调银行，不落支付记录或落 REJECTED）
   ├─ 3. 幂等检查（同 key 返回原结果）
   ├─ 4. 落库 PENDING
   ├─ 5. 调银行模拟器（带超时）
   │      ├─ approved ──▶ 更新 AUTHORIZED ──▶ 200 Authorized
   │      ├─ declined ──▶ 更新 DECLINED   ──▶ 200 Declined
   │      └─ 超时或 5xx ──▶ 保持 PENDING   ──▶ 504 / 202，不说 Declined
   └─ 日志只记 last4、payment_id，不记卡号和 CVV
```

## 6. 代码（容易读的版本，Spring Boot）

```java
public record PaymentRequest(
        String cardNumber, Integer expiryMonth, Integer expiryYear,
        String cvv, Long amount, String currency) {}

public record FieldError(String field, String code, String message) {}

public class PaymentValidator {
    private static final Set<String> SUPPORTED_CURRENCIES = Set.of("GBP", "USD", "EUR");

    public List<FieldError> validate(PaymentRequest r, YearMonth now) {
        List<FieldError> errors = new ArrayList<>();
        validateCardNumber(r.cardNumber(), errors);
        validateExpiry(r.expiryMonth(), r.expiryYear(), now, errors);
        validateCvv(r.cvv(), errors);
        validateAmount(r.amount(), errors);
        validateCurrency(r.currency(), errors);
        return errors;
    }

    private void validateCardNumber(String pan, List<FieldError> errors) {
        if (pan == null || pan.isBlank()) {
            errors.add(new FieldError("card_number", "REQUIRED", "Card number is required"));
            return; // 缺失就不再报别的错
        }
        if (!pan.matches("\\d{13,19}")) {
            errors.add(new FieldError("card_number", "INVALID_FORMAT", "Must be 13-19 digits"));
        } else if (!passesLuhn(pan)) {
            errors.add(new FieldError("card_number", "INVALID_LUHN", "Card number is not valid"));
        }
    }

    private void validateExpiry(Integer month, Integer year, YearMonth now, List<FieldError> errors) {
        if (month == null || month < 1 || month > 12) {
            errors.add(new FieldError("expiry_month", "OUT_OF_RANGE", "Month must be 1-12"));
            return;
        }
        if (year == null) {
            errors.add(new FieldError("expiry_year", "REQUIRED", "Year is required"));
            return;
        }
        if (YearMonth.of(year, month).isBefore(now)) { // 当月仍然有效
            errors.add(new FieldError("expiry_year", "EXPIRED", "Card has expired"));
        }
    }

    private void validateCvv(String cvv, List<FieldError> errors) {
        if (cvv == null || !cvv.matches("\\d{3,4}")) {
            errors.add(new FieldError("cvv", "INVALID_FORMAT", "CVV must be 3 or 4 digits"));
        }
    }

    private void validateAmount(Long amount, List<FieldError> errors) {
        if (amount == null || amount <= 0) {
            errors.add(new FieldError("amount", "INVALID", "Amount must be a positive integer (minor units)"));
        }
    }

    private void validateCurrency(String currency, List<FieldError> errors) {
        if (currency == null || !SUPPORTED_CURRENCIES.contains(currency)) {
            errors.add(new FieldError("currency", "UNSUPPORTED", "Currency is not supported"));
        }
    }

    private boolean passesLuhn(String pan) {
        int sum = 0;
        boolean doubleIt = false;
        for (int i = pan.length() - 1; i >= 0; i--) {
            int d = pan.charAt(i) - '0';
            if (doubleIt) {
                d *= 2;
                if (d > 9) d -= 9;
            }
            sum += d;
            doubleIt = !doubleIt;
        }
        return sum % 10 == 0;
    }
}

@RestController
public class PaymentController {
    private final PaymentValidator validator;
    private final BankClient bank; // 接口，测试里可以替换成 mock

    @PostMapping("/payments")
    public ResponseEntity<?> pay(@RequestHeader("Idempotency-Key") String key,
                                 @RequestBody PaymentRequest request) {
        List<FieldError> errors = validator.validate(request, YearMonth.now());
        if (!errors.isEmpty()) {
            return ResponseEntity.unprocessableEntity()
                    .body(Map.of("status", "Rejected", "errors", errors));
        }
        try {
            BankResult result = bank.authorize(request); // 带超时
            return ResponseEntity.ok(toResponse(result)); // Authorized 或 Declined
        } catch (BankTimeoutException e) {
            return ResponseEntity.status(504).body(Map.of("status", "Pending"));
        }
    }
}
```

代码说明（面试时可以讲）：
- 校验器是**纯函数**，传入当前时间，容易测试，不依赖 Spring。
- `BankClient` 是接口，测试时换成 mock，可以覆盖 approved、declined、timeout。
- 金额用 `long` 最小单位，不用 `double`。
- 日志里只记 `payment_id` 和 last4。
- 这版省略了：幂等存储、落库、真正的卡号类型（见 `card-vault-design.md` 的 `CardNumber`）。要说出来，不要假装没有。

## 7. 测试清单

| 类别 | 用例 |
| --- | --- |
| 有效 | 合法请求 -> 调银行一次 |
| 单个错误 | Luhn 不对、月份 13、已过期、金额 0、币种不支持 |
| **多个错误** | 同时有 3 个错误 -> 返回 3 条，**银行 mock 调用次数为 0** |
| 边界 | 有效期是当月（有效）、上个月（过期）、卡号 13 位和 19 位、CVV 3 位和 4 位 |
| 银行结果 | approved、declined、超时（不是 Declined） |
| 安全 | 响应和日志里没有完整卡号或 CVV |
| 重复请求 | 同一个 Idempotency-Key 返回同样的结果，银行只被调一次 |

## 8. 英文口述（约 1 分钟）

"I split checks by who has the information. The gateway validates what it can know from the request alone: required fields, card number format and Luhn, expiry, CVV format, positive amount in minor units, and supported currency. These are cheap and stable, so I fail early and never call the bank for them. The bank decides what only it can know: funds, blocked cards, CVV match, fraud and 3DS. I don't copy those rules into the gateway, because they change and differ by bank. For errors, I collect all validation failures and return them in one 422 response with a field, a stable code and a message, so the merchant can fix everything at once. Declined is a normal business result, so it returns 200. A bank timeout is neither Declined nor Rejected; I keep it pending, because I don't know whether the bank charged. Does this match what you had in mind?"

## 9. 面试官可能追问

| 问题 | 回答要点 |
| --- | --- |
| 为什么 Declined 用 200，Rejected 用 422？ | Declined 是处理成功的业务结果；Rejected 是调用方请求有问题 |
| 为什么收集所有错误？ | 商户一次改完，少来回；代价是校验器稍复杂，字段间依赖要处理 |
| 银行超时怎么办？ | 不是 Declined；保持 PENDING，用同一个引用查询或重试，不重复扣款 |
| Luhn 通过就说明卡有效吗？ | 不，只是防输错，卡是否真实有效由银行判断 |
| 为什么有效期当月算有效？ | 卡到当月最后一天才过期 |
| 要不要把 Rejected 的请求存下来？ | 可以存审计记录（不含卡号），方便排查；但不要占用幂等键，除非设计需要 |
| 校验规则变多怎么办？ | 每个检查一个小方法或规则对象，加规则不改老代码（开闭原则） |

## 10. 怎么拆任务：一步一步让 AI 写，每步配测试

原则：**每一步都能单独运行和测试，通过了再进入下一步**。每步的节奏固定：说计划 -> 写 prompt -> 审查输出 -> 跑测试 -> 提交。

### 10.1 拆分顺序（由内到外，先纯逻辑，后接口）

| 步骤 | 做什么 | 为什么这个顺序 |
| --- | --- | --- |
| 1 | 定义数据类型：`PaymentRequest`、`FieldError`、`PaymentStatus`、`BankResult` | 后面都依赖它，先定下契约 |
| 2 | `PaymentValidator`（纯函数，收集全部错误） | 没有外部依赖，最容易测，也是核心规则 |
| 3 | `BankClient` 接口 + 模拟器（或 fake） | 先定接口，才能 mock |
| 4 | `PaymentService`：校验 -> 调银行 -> 映射结果 | 业务编排，依赖前两步 |
| 5 | `PaymentController`：`POST /payments`，HTTP 状态码、响应格式 | 最外层，只做转换 |
| 6 | 超时和异常处理（银行超时 -> Pending，不是 Declined） | 在主流程通了之后再加 |
| 7 | 补强：幂等键、日志脱敏、并发（时间够再做） | 加分项，说清楚没做的也行 |

为什么先内后外：内层是纯逻辑，测试快、定位简单；等到接口层，出问题就知道是 HTTP 的问题，不是规则的问题。

### 10.2 每一步的 prompt 和测试

**步骤 1：数据类型**

```text
Prompt: Create Java records: PaymentRequest(cardNumber, expiryMonth, expiryYear, cvv, amount, currency),
        FieldError(field, code, message), and an enum PaymentStatus {AUTHORIZED, DECLINED, REJECTED, PENDING}.
        Amount is a long in minor units. Records only, no logic.
测试: 不需要。审查：金额是不是 long？字段有没有漏？
```

**步骤 2：校验器（重点）**

```text
Prompt: Write PaymentValidator.validate(request, YearMonth now) returning List<FieldError>.
        Rules: card number 13-19 digits and passes Luhn; month 1-12; expiry not before current month;
        cvv 3-4 digits; amount > 0; currency in GBP/USD/EUR.
        Collect ALL errors, do not stop at the first. If a field is missing, report only "REQUIRED" for it.
        Never put the card number in a message. Small private methods, easy to read.
```

先写测试（可以自己写，或让 AI 写后逐条检查）：

| 测试 | 期望 |
| --- | --- |
| 合法请求 | 返回空列表 |
| Luhn 不对 | 1 个错误，`INVALID_LUHN` |
| 月份 13 | `OUT_OF_RANGE` |
| 有效期是当月 | 通过（边界） |
| 有效期是上个月 | `EXPIRED` |
| 金额 0 和负数 | `INVALID` |
| 币种 XYZ | `UNSUPPORTED` |
| **同时 3 个错误** | 返回 3 条（证明不是 fail-first） |
| 卡号为 null | 只有 `REQUIRED`，没有 Luhn 错误 |
| 错误信息 | 不含完整卡号 |

**步骤 3：银行接口和模拟器**

```text
Prompt: Create interface BankClient { BankResult authorize(PaymentRequest r); }.
        BankResult has status APPROVED/DECLINED and a declineCode.
        Throw BankTimeoutException on timeout.
        Then a FakeBankClient for tests that I can configure to approve, decline, or time out,
        and that counts how many times it was called.
```

测试：fake 本身行为正确（可简单验证）。**调用次数计数是后面"Rejected 时不调银行"的关键。**

**步骤 4：PaymentService**

```text
Prompt: PaymentService.process(request): validate; if errors, return Rejected without calling the bank;
        otherwise call BankClient and map APPROVED -> Authorized, DECLINED -> Declined.
        On BankTimeoutException return Pending, not Declined.
```

| 测试 | 期望 |
| --- | --- |
| 有效请求 + 银行同意 | Authorized，银行调用 1 次 |
| 有效请求 + 银行拒绝 | Declined，带 decline code |
| **无效请求** | Rejected，**银行调用 0 次** |
| 银行超时 | Pending，不是 Declined |
| 多个错误 | Rejected 里有全部错误 |

**步骤 5：Controller**

```text
Prompt: POST /payments returning 422 with {status, errors} for Rejected, 200 for Authorized/Declined,
        504 with {status: "Pending"} for timeout. Response contains last4 only, never the full card number.
```

测试（`MockMvc` 或 `@WebMvcTest`）：状态码、JSON 结构、响应里**没有**完整卡号。

**步骤 6：补强**
- 日志脱敏：加测试，捕获日志，断言不含 16 位数字。
- 幂等键：同一个 key 调两次，银行只被调 1 次。
- 说出没做的："Persistence is in-memory here; in production I'd store the PENDING operation first."

### 10.3 每一步的固定节奏（要说出来）

```text
1. 说计划   "Next I'll build the validator. It's pure logic, so easy to test."
2. 写 prompt "I'll ask the AI for this one class only, with these rules."
3. 审查输出 "It used double for amount / forgot null check. I'll fix it."
4. 跑测试   "Let me add the three-errors test first. It should fail if the code stops at the first error."
5. 小结     "Validator done. Now the bank client interface."
```

### 10.4 技巧

- **测试先行也可以**：先让 AI 或自己写失败的测试，再写实现。展示你用测试驱动。
- **让 AI 写测试后，你要检查**：它常常只测 happy path，要补边界和失败。
- **故意问一次"这个测试会在什么情况下失败？"**，防止写出永远通过的测试。
- **时间不够**：保证 1–2–4–5 四步和对应测试，其余口头说明。
- **每步后跑一次全部测试**，不要攒到最后。

### 10.5 英文口述

"I'll build this from the inside out. First the data types, then the validator, which is pure logic, so I can test it thoroughly. Then a bank client interface with a fake, so I can count calls. Then the service that wires them together, and finally the controller. At each step I give the AI one small task with clear rules, review the result, and write tests before moving on. The most important test is that an invalid request is rejected with all its errors and the bank is called zero times."

## 11. 安全追问：讲清楚原理（面试官最爱问）

### 11.1 Gateway 怎么知道是哪个商户在调用？（认证 Authentication）

**认证 = 你是谁。授权 = 你能不能碰这个资源。** 两步，顺序不能反。

```text
 商户注册时：我们生成 API key，例如 sk_live_9fA3...（只在创建时显示一次）
              数据库里只存 key 的哈希（SHA-256），不存明文
              同时记下：这个 key 属于哪个 merchant_id

 每次请求：   Authorization: Bearer sk_live_9fA3...
   Gateway：  1. 取出 key，算哈希
              2. 用哈希查库 -> 找到 merchant_id（找不到 -> 401）
              3. 把 merchant_id 放进请求上下文，后面所有代码都用它
```

要点（为什么这样做）：

| 做法 | 原因 |
| --- | --- |
| 只存哈希，不存明文 | 数据库泄露时，攻击者拿不到可用的 key（和密码同理） |
| `merchant_id` 来自**认证结果**，**不来自请求体或 URL** | 否则商户可以随便填别人的 id |
| key 带前缀 `sk_live_` / `sk_test_` | 一眼分清环境，泄露后容易发现、扫描 |
| 支持轮换（新旧 key 同时有效一段时间） | key 泄露时可以换，又不停服务 |
| 限流 | 防暴力猜 key |

更强的方式（被问到再说）：OAuth2 client credentials（短期 token）、mTLS（双向证书）、请求签名（HMAC）。**API key 是起点，简单够用（KISS）。**

### 11.2 商户 A 怎么读不到商户 B 的支付？（授权 Authorization，防 IDOR）

**IDOR（Insecure Direct Object Reference）：** 接口用 id 直接查数据，但没检查这个 id 是不是调用者的。

```text
 错误：  GET /payments/pay_123  ->  SELECT * FROM payments WHERE id = 'pay_123'
         商户 A 猜到 B 的 pay_123，就能读到。

 正确：  SELECT * FROM payments WHERE id = 'pay_123' AND merchant_id = :当前商户
         不属于当前商户 -> 查不到 -> 返回 404
```

要点：
- 返回 **404，不是 403**。403 等于告诉对方"这笔支付存在，只是你不能看"，泄露信息。
- **每一个**按 id 查询的接口都要带 `merchant_id`，不只是 GET，还有退款、取消。
- 幂等键也按商户隔离：唯一约束 `(merchant_id, idempotency_key)`。否则 A 的 key 会和 B 的冲突，甚至读到 B 的响应。
- id 用随机值（不是自增 1,2,3），增加猜中的难度；但**这只是额外一层，真正的保护是 merchant_id 检查**。

> "I authenticate with an API key whose hash is stored, and the merchant id comes from the authenticated identity, never from the request body. Every query by payment id also filters by that merchant id, and I return 404 for payments that belong to someone else."

## 12. 扩展性追问：讲清楚原理

先记住一句话：**扩展性 = 找到先撑不住的那个点，再针对它处理。** 不要一上来就说"加机器、分库"。

### 12.1 先看整条链路，哪里是瓶颈

```text
 商户 -> [Gateway/Payment Service] -> [数据库] -> [银行]
          无状态，容易扩        有状态，难扩    外部，不受我们控制
```

| 位置 | 为什么容易/难 | 处理 |
| --- | --- | --- |
| Gateway / Payment Service | **无状态**：不保存任何本机状态，哪台机器处理都一样 | 前面放负载均衡，**加机器（水平扩展）**即可 |
| 数据库 | **有状态**：数据只有一份，写入要保证一致 | 最难扩，是第一个瓶颈（12.2） |
| 银行 | 外部服务，慢、会限流，我们改不了 | 隔离和保护（12.3） |

**无状态**是什么意思：服务不在内存里保存用户会话或中间结果，状态都在数据库里。这样任意一台机器挂了，请求打到另一台也能处理。

**水平扩展 vs 垂直扩展：** 水平 = 加更多机器；垂直 = 换更大的机器。垂直有上限且贵，水平靠无状态才能做。

### 12.2 数据库怎么扩（重点，必须能讲清楚）

我们的写入模式：每笔支付 **至少两次写**（先插入 PENDING，再更新结果）。流量 10 倍，写入也 10 倍。

**第一步：先别分库，先优化**
- 加合适的索引（查询路径：`(merchant_id, idempotency_key)`、`payment_id`）。
- 缩短事务（不要在事务里等银行响应，见下）。
- 升级机器（垂直扩展，最简单）。

**第二步：读写分离（Read Replica）**
```text
 写 -> 主库 (Primary) ---复制---> 从库 (Replica) <- 读（查询、报表）
```
- 解决"读多"的问题，**不解决"写多"**。
- **复制有延迟**：刚写完马上去从库读，可能读不到。所以"支付后立刻查状态"这类读，要读主库（read-your-writes）。报表、历史查询可以读从库。

**第三步：分片（Sharding）**
- 把数据按某个 **分片键（shard key）** 拆到多台数据库，每台只存一部分。
- 为什么选 `merchant_id`：
  - 大多数查询都带商户，请求只落到一个分片，不用跨库。
  - 幂等的唯一约束 `(merchant_id, key)` 也在同一个分片内，仍然有效。
- 方式：哈希（`hash(merchant_id) % N`，均匀）或范围；常用**一致性哈希**或"逻辑分片 + 映射表"，方便以后加机器时少搬数据。
- **代价（面试要讲）：**
  - **热点**：一个超大商户的流量集中在一个分片。对策：大商户单独分片，或用 `merchant_id + payment_id` 组合分散。
  - **跨分片查询**很麻烦（比如"全平台某天的汇总"）：用报表库或数据仓库来做，不在交易库里查。
  - **扩容要搬数据**（resharding），要提前设计。
  - 跨分片事务基本不要做，设计时让一笔支付的所有数据在同一分片。

**为什么不在事务里等银行：** 事务一直占着数据库连接和行锁。银行慢（比如 5 秒）时，连接池耗尽，整个服务卡死。所以：
```text
 事务 1（很短）：插入 PENDING -> 提交
 事务外：调银行（可能很慢）
 事务 2（很短）：更新结果 -> 提交
```

> "The first bottleneck is the database write path. I'd first add indexes and keep transactions short, never holding one open during the bank call. Then replicas for reads, with read-after-write queries going to the primary. If writes still outgrow one node, I'd shard by merchant id, so most queries and the idempotency unique constraint stay within one shard. The trade-offs are hot merchants, cross-shard reporting, and resharding."

### 12.3 银行调用怎么保护

问题：银行慢时，我们的线程一直在等。线程池被占满 -> 其他银行的请求也进不来 -> 整个服务卡死。这叫**级联故障**。

| 机制 | 是什么 | 解决什么 |
| --- | --- | --- |
| **超时（Timeout）** | 等银行最多 N 秒，到了就放弃 | 不会无限等待 |
| **舱壁（Bulkhead）** | 每家银行**单独的线程池/连接池** | 一家慢，不会占光所有资源，其他银行正常 |
| **熔断（Circuit Breaker）** | 某家银行连续失败超过阈值，**暂时不再调它**（快速失败），过一会儿放少量请求试探，恢复了再打开 | 给银行喘息时间，也不让我们的线程白等 |
| **限流（Rate Limiting）** | 每个商户、每家银行有请求速率上限 | 一个商户不会拖垮其他人，也不超过银行的限额 |
| **退避重试（Backoff）** | 重试间隔越来越长，加随机抖动 | 避免大家同时重试，把银行打垮 |

熔断三个状态：
```text
 Closed（正常）--失败太多--> Open（拒绝，快速失败）--等一段时间--> Half-Open（试探几个请求）
      ^                                                                  |
      +----------------------成功，恢复--------------------------------+
```

### 12.4 其他常见手段（被问到再用）

| 手段 | 什么时候用 | 取舍 |
| --- | --- | --- |
| **缓存（Cache）** | 读多、变化少的数据：商户配置、汇率、路由规则 | 要有明确的失效规则；**不缓存会变的支付状态**；**绝不缓存卡号** |
| **消息队列** | 非关键路径：webhook 通知、对账、报表。让它们和主流程解耦 | 增加复杂度，要处理重复和顺序；授权这种要立刻返回结果的，**不能**异步 |
| **异步 + 轮询/webhook** | 银行响应很慢时，先返回 Pending | 调用方要处理最终状态 |
| **多区域部署** | 低延迟、容灾 | 数据一致性复杂，面试里先说"单区域 + 可恢复"，再说多区域 |

### 12.5 流量 10 倍的回答模板

1. **先说怎么发现：** 看指标：p99 延迟、数据库写延迟、连接池使用率、银行错误率和超时率。
2. **按瓶颈顺序处理：** 无状态服务加机器 -> 数据库（索引、短事务、读写分离、分片）-> 银行（超时、舱壁、熔断、限流）。
3. **说取舍：** 分片要考虑热点、跨分片查询、扩容；缓存要考虑失效和安全。
4. **说不做什么（YAGNI）：** 流量没到之前不上分片，保持简单。

> "I'd first check metrics to find the real bottleneck. The stateless services scale by adding instances. The database is the first real limit, so I'd handle it in steps: indexes and short transactions, then replicas, then sharding by merchant id if needed. For the banks, I'd use timeouts, a bulkhead per bank and a circuit breaker so one slow bank can't take everything down. I wouldn't shard before the numbers justify it."

## 13. 第 3 轮面试的准备方向

HR 确认：**第 2 轮是线下作业，做完后找面试官 review；第 3 轮是 system design，重点会有可扩展性。** 所以：
- 第 2 轮：按 §10 的步骤提前做一遍，准备好"为什么这样设计"（安全、可扩展、测试）的解释，review 时要能把每个决定讲清楚。
- 第 3 轮：需要对 high-level 概念有真正的理解。重点概念见 §12，其他主题在 `payment-processor-design-prep.md` 第 6 章（银行路由、慢银行隔离、限流、数据库扩展、对账、安全、灾备）。
