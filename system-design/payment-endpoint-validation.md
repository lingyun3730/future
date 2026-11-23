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
