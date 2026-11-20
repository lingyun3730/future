# InHouse Buyer FX: Interview Story and Follow-up Questions

本文用于英文面试练习，基于本人提供的项目经历整理。英文口述与中文准备笔记分开；未确认的机制、指标和个人编码范围明确标注，不作为既成事实。

> 整理说明：本文由笔记截图转录而成，缺失的截图内容（原 1 节表格末尾之后的少量内容、第 3 节中间过渡段、第 4 节表格后的部分说明、第 5 节问题 2 的英文回答、问题 4/5 的英文回答、问题 14 的结尾）已标注“待补”，请对照原稿补全。

## 1. 故事主线

降低 FX 成本 → 梳理账户与跨系统迁移场景 → 解决生产验证覆盖问题 → 按流量特征选择控制策略 → 完成验证。

| 要素 | 已知内容 |
| --- | --- |
| 业务背景 | 原先 processor 提供 Buyer FX，并在汇率中收取 markup |
| 目标变化 | Processor 按买家支付币种收单，eBay 直接与银行做 FX trades，以降低 FX 成本 |
| 个人责任 | 项目 tech lead：理解端到端现状、厘清 case studies、识别系统和团队修改范围，协调开发、E2E 测试与上线；亲自写代码（FX team 负责 FXCore 中计算 FX data points 的部分，有代码改动和测试） |
| Processor 背景 | Adyen 是 eBay 最早合作的 card processor；CKO 是后来集成的 |
| 难点一 | Processor 的币种与 settlement 支持不同；部分 merchant accounts 被 Buyer FX 和 domestic transactions 复用，迁移需要新账户等调整 |
| 难点二 | 无法自行生成覆盖全部 currency pairs 的生产测试订单，也没有预算使用第三方机构 |
| 关键方案 | Token-based traffic controller：高频 pair 事前预扣，低频 pair 允许多实例捕获后再扣减 |
| 已确认结果 | 按流量将 currency pairs 分组，完成范围内所有币种对（33 种 payment currencies × 7 种 listing currencies）的 smoke testing，由本人用一周完成，验证 processor 的 settlement 行为 |
| 失败恢复 | Token 额度不自动恢复；领取后交易失败的，人工检查后再补额度 |

不要把 InHouse Buyer FX 与 Dynamic Pricing、Payment Currency Expansion 混成一个项目，也不要把后者的 42 个新增国家和地区作为本项目结果。

### 1.1 口述提示卡：看关键词，不背全文

每行用自己的话讲 1–2 句。先讲主线，被追问再展开细节。

| 顺序 | English cues | 想起什么 |
| --- | --- | --- |
| 背景与目标 | processor FX → markup → lower costs → banks | 原来 processor 换汇有加价；希望 eBay 直接与银行交易 |
| 我的工作 | full payment flow → changes → teams → launch | 我梳理端到端链路与改动，推动各团队开发、测试和上线 |
| 第一个难点 | different currencies → shared accounts → new accounts | 各 processor 能力不同，旧账户复用，不能简单改一个配置 |
| 第二个难点 | every currency pair → production testing → no budget | 要验证真实 settlement，但没有覆盖全部币种对的测试订单和第三方预算 |
| 方案 | live traffic → traffic controller → tokens | 用 token 控制少量真实交易进入新路径 |
| 取舍 | busy pairs: deduct first / rare pairs: deduct later | 高频先扣额度；低频让多个 worker 捕获后扣，可能超过一笔 |
| 结果 | all pairs in scope → smoke tests → expected settlement | 完成范围内各币种对验证；不编节省数字 |

只记一条线：cost → my role → accounts → testing → tokens → two modes → result。

卡住时用：“Before the project...” / “My job was...” / “The first challenge was...” / “So we...” / “In the end...”。

## 2. 30-second Overview

> I led a project to bring Buyer FX in-house at eBay. We wanted to lower FX costs by trading directly with banks instead of using processors for currency conversion. My job was to work out what needed to change and help the teams deliver it. One challenge was testing settlement for each currency pair in production. We used a traffic controller to test a small amount of live traffic.

## 3. Main Story: Simple Spoken English

> One project I led was bringing Buyer FX in-house at eBay.
>
> Before this project, processors handled currency conversion for us. They added a markup to their rates, which increased our FX costs. We wanted processors to collect payments in the buyer's currency, while eBay traded directly with banks.
>
> My job was to understand the full payment flow and work out what each system needed to change. I then worked with the teams on development, testing, and launch.
>
> There were two main challenges.
>
> First, we couldn't just change one setting. Different processors supported different currencies and ways of settling payments. Some merchant accounts were used for both Buyer FX and domestic payments. I went through these cases to work out where we needed new accounts and changes in other systems.
>
> Second, we needed to check that each currency pair settled as expected in production. We couldn't create test orders for every pair, and we didn't have the budget to pay a third party to do it.
>
> So we used a traffic controller to send a small amount of live traffic through the new setup. It used tokens to control which transactions could enter the test path.
>
> Some pairs had lots of traffic. Others might have no transactions for days. For busy pairs, a worker took and deducted the token before using the new path. For rare pairs, several workers could look for a matching transaction and deduct the token after processing it. This gave us a better chance of finding rare transactions, but it could let more than one through.
>
> We chose the mode based on each pair's traffic. In the end, we finished smoke testing for all the pairs we needed to test and checked that the processors settled them as expected.

讲述提示：

- I 用于本人分析、判断和推动的工作；we 用于团队方案与实现。不把 Traffic Controller 的全部代码归为个人实现。
- 核心不是“预算不足所以去生产试错”，而是怎样以受控真实流量补足验证覆盖。
- 用简单短句；被追问时先回答当前问题，不强行背完全文。
- 业务目标是降低 FX 成本；不编造节省比例、收入增量或零故障结论。

## 4. Technical Deep Dive: Two Control Modes

| 场景 | 做法 | 目的与代价 |
| --- | --- | --- |
| 高频 currency pairs | Pessimistic mode：worker 获取 token 时立即预扣额度 | 优先控制进入 smoke test 路径的交易数量；领取后失败可能消耗额度，实际恢复方式待确认 |
| 低频 currency pairs | Optimistic mode：多个 worker 可以看到目标 token，捕获处理后再扣减 | 提高捕获稀有交易的机会；可能有多个并发交易进入，不是严格的一笔上限 |

可用于英文展开：

> For busy pairs, we wanted to keep the number of test transactions low. For rare pairs, we wanted more workers to have a chance to find a match. The second approach could let more transactions through than the token count, so it wasn't a strict limit.

准确性边界：每个币种对先验证一笔是业务目标，不代表两个模式都能保证 exactly one transaction。Pessimistic mode 的严格保证还依赖并发领取与扣减的原子性，以及接入侧如何使用 token。不要仅凭模式名称就宣称端到端 exactly-once。

现有 `interview-preparation.md` 的 Traffic Controller 附录记录了服务端 CAS 核对及其限制，可作为进一步准备的参考；本故事不新增源码核对结论，也不把尚未核实的客户端、失败恢复或 CAS 完整语义写成事实。

## 5. Follow-up Questions

### Business Context and Ownership

**1. "Why did you bring Buyer FX in-house?"**

回答重点：降低 processor-managed conversion 的 FX 成本，由 eBay 与银行做 trading。不要说“取消所有 FX 成本”。

> We wanted to lower the FX costs charged by processors. This changed who handled the conversion and trading. It wasn't just a change to how we looked up rates.

**2. "Why wasn't this just a configuration change?"**

回答重点：不同 processor 的币种支持和 settlement 能力不同，包括是否支持 L4L settlement；部分旧账户与 domestic transactions 复用，需要逐场景……（待补：截图此处被截断。）

待补：一个可公开的具体 case，解释原方案为什么不适用、迁移需要哪些修改，不暴露真实账户标识。

**3. "What was your part, and what did the other teams do?"**

> I worked out what needed to change across the payment flow and which teams needed to be involved. I then helped the teams deliver those changes through development, testing, and launch.

待补：一个本人提出或推动的具体技术决策、本人直接编码的范围，以及其他团队的责任。避免只讲会议和进度协调。

### Production Validation and Trade-offs

**4. "Why did you need production smoke testing?"**

回答重点：需要验证每个目标 currency pair 的真实 settlement 行为，但无法生成覆盖全部币种对的订单，也没有第三方验证预算。受控生产验证是补充，不替代前置 E2E 测试。

**5. "Why not just sample a small percentage of traffic?"**

回答重点：百分比不能直接保证逐币种对覆盖；高频 pair 可能进入很多笔，而低频 pair 可能一直没有样本。目标币种对选择与 token 额度控制更贴合验证需求。

这是解释方案合理性的对比，不代表团队当时一定正式评估并否决过百分比采样。

**6. "Why did you introduce two modes?"**

> Busy pairs needed tighter control, so we deducted the token first. Rare pairs needed more chances to find a transaction, so several workers could look for a match and deduct the token afterwards.

回答重点：按实际流量分布取舍，不是简单的“一个快、一个慢”。只有一个实例持有低频 token 时，目标交易可能落到其他实例。

**7. "Can you guarantee exactly one transaction per currency pair?"**

> Not with the optimistic mode. Several workers can see the same token before it's deducted, so more than one transaction may enter. The pessimistic mode deducts the token first, which gives us tighter control.

回答重点：明确并发窗口与取舍，不声称两种模式都有严格的一笔上限。若进一步解释 pessimistic 的保证，需要说明实际原子领取机制及接入语义。

**8. "What happened if a worker failed after reserving a token?"**

待确认：是否归还、过期、人工补发，或采用其他恢复方式。领取 token 不代表交易已经完成，交易完成也不代表 settlement 已验证成功。不要把建议的恢复设计说成当时实现。

**9. "Is this a standard token-bucket rate limiter?"**

> Here, a token lets a worker use the test path. It's not a standard token bucket that refills automatically.

回答重点：这是按目标场景配置和消费的准入额度，不能假设存在固定速率补充。Traffic Controller 不等于直接代理所有支付请求。

**10. "What exactly did you validate, and how did you know it was correct?"**

回答重点：已确认的是逐 currency pair 验证 processor 的 settlement 是否符合预期，而不是只看支付 API 成功。

待补：实际检查的币种、金额或其他核对项，预期结果的来源，使用的报表或记录，以及确认责任人。只有实际验证过的内容才能加入英文答案。

**11. "Does one successful transaction prove the migration is safe?"**

> No. It shows that the basic flow works for that pair. It doesn't prove that every failure case works or that the system can handle full traffic.

回答重点：区分 smoke testing、异常场景验证与规模验证。后续放量阈值、监控和回滚的实际步骤待补；不虚构自动回滚机制。

### Delivery, Results, and Reflection

**12. "How did you coordinate the work across teams?"**

回答重点：先厘清 case studies、账户结构和系统变化，再明确团队责任及依赖，推动开发、测试与上线。

待补：一次真实的依赖阻塞或意见分歧，你采取了什么行动，结果如何。当前没有足够事实编写冲突故事。

**13. "What was the result, and how did you measure it?"**

> We finished smoke testing for all the pairs we needed to test and checked that the processors settled them as expected.

回答重点：先给出已确认结果。成本降低是项目目标，若引用实际节省金额、比例或测量方法，需另有可靠依据；没有数字就明确说没有可引用的具体数据。

**14. "What would you do differently next time?"**

待本人确认的方向：是否应更早建立 currency-pair 覆盖矩阵、分析流量分布、明确验证证据，或设计 token 失败恢复。选择真实体会，不为满足行为题而编造失误。

可选总结，仅在符合本人实际反思时使用：

> I learned that we should plan production testing as part of the migration, not leave it until the end. We also needed different controls for busy and （待补：截图在此处被截断。）

## 6. Useful Expressions

| English | 中文与使用边界 |
| --- | --- |
| Bring Buyer FX in-house | 将原由 processor 承担的能力转到内部 |
| Exchange-rate markup | 汇率加价，不等于所有 FX 成本 |
| Like-for-like settlement | 按相同币种结算，具体支持依 processor 和账户配置说明 |
| Merchant-account mapping | 商户账户配置映射，不是给卖家选择银行账户 |
| Currency pair | 币种对；准备好解释本项目中两个币种各自的含义 |
| Reserve a token upfront | 事前领取并预扣额度 |
| Consume the token after processing | 事后扣减，不是严格事前限额 |
| Control production exposure | 控制进入新链路的生产交易范围与数量 |
| Validate settlement behavior | 验证结算行为，不只看接口成功 |
| End-to-end technical ownership | 对端到端技术分析和交付负责，不代表所有代码由本人实现 |

## 7. 下一轮需要补齐的事实

- 一个具体 merchant-account 迁移 case，以及本人作出的判断。
- 本人直接设计、实现与协调的边界。
- Token 存储、原子领取、失败恢复与接入侧消费的实际机制。
- 每笔 smoke transaction 的验证证据与成功标准。
- 实际放量、监控与回滚步骤。
- 一个跨团队协作事件和一条真实反思。

练习顺序：30 秒概述 → 完整故事 → 优先回答问题 3、6、7、10、13。记住节点和取舍，不逐字背诵。

## 6. Coding principles with examples（第一轮用）

| 原则 | 我的例子 |
| --- | --- |
| 先理解全链路，再动手 | 先梳理端到端流程和 case studies，找出共享 merchant account、需要新账户等风险，再开发 |
| 覆盖不了就受控地验证 | 无法造出全部币种对的测试单，用 Traffic Controller 让少量真实交易进入新路径；33 × 7 个币种对一周完成 smoke test |
| 宁可少用，不要多用（安全失败） | Token 额度不自动恢复，人工核实后再补，避免失败后重复放量 |

> One principle I follow is to control risk when I can't test everything. In Buyer FX, we couldn't create test orders for every currency pair, so we used a traffic controller to let a small amount of live traffic through the new path. If a transaction failed, the token was not restored automatically. A person checked it first, because we preferred under-testing to sending too much traffic.

注意：Pessimistic 与 Optimistic 两种模式的严格保证边界见第 4 节，不要说成 exactly-once。
