# First Round Prep: Coding Principles, Hard Problems, Incidents, Testing

第一轮面试的四个主题放在一起准备：
1. Coding principles（带例子）
2. 技术难点 / 把项目推进成功
3. 真实的 production incident / bug
4. Unit test、CI/CD（以及可测试性、可观测性）

核心思路：**原则是线，故事是点。** 先用一句话讲原则，再用一个真实例子撑住，然后把话头交给面试官，让他决定要不要深挖。不要一口气讲完整个故事。

`[待补]` = 需要我补真实情况，不要编。详细故事在 `interview-cards.md`，测试和 CI/CD 的问答在 `testing-ci-observability-qa.md`。

---

## 1. How to answer naturally

### The pattern (每个回答 30–60 秒)

> **Principle** (one sentence) → **Example** (2–3 sentences: what happened, what I did) → **Result or lesson** (one sentence) → **Offer** ("Happy to go deeper into X if you like.")

- 先给结论，再给例子，不要先讲背景。
- 讲完一个例子就停，让面试官追问。追问时再展开（卡片里的 90 秒版和追问答案）。
- 一个问题只用一个故事。同一轮里同一个完整故事只讲一次，换角度用一句话带过。
- 不确定面试官想要什么，就先问一句："Would you like a short example from my work?"

### Useful bridges (自然的衔接语)

- "A good example of that is..."
- "That's similar to something I faced in..."
- "What I learned from that was... and I applied it later in..."（把两个故事串起来，例如 FX 小数位事故 → 币种扩展的 E2E）
- "I can go deeper on the design or on how I handled the incident, whichever you prefer."

---

## 2. Coding principles (with examples)

### The 8 principles at a glance

| # | Principle | One-line meaning | My example (one line) | Full story |
| --- | --- | --- | --- | --- |
| 1 | **Fail early** | Catch bad input at the earliest point | EasyFin validates at submission, so a wrong journal type (BFX vs SFX) is caught before anyone reviews it | EasyFin |
| 2 | **Design for failure** | Assume things fail: limit damage, keep a manual override, prefer the safe failure | TasCreed: exponential backoff for endless retries, plus a kill API. Buyer FX: tokens are not restored automatically, a person checks first | TasCreed, Buyer FX |
| 3 | **Protect the downstream** | Don't let every request hit the service behind you | FXCenter client reads rates from a local cache. Incomplete data: use the previous day's full set, no retry loop, raise an alert | FXCenter |
| 4 | **Safe to re-run (idempotent)** | The same work done twice gives the same result | TasCreed runs tasks at least once. Each shard writes a fixed file name, so a re-run overwrites it | TasCreed |
| 5 | **Fix at the root, stay backward compatible** | Fix the shared cause, and don't break existing callers | TasCreed: fixed the framework, not each report. Shared function: add a new function, leave the old one untouched | TasCreed, shared function |
| 6 | **Verify with evidence, test every flow** | When unsure, test or control the risk; test every flow that uses the data | Currency expansion: tests showed 25 currencies not configured. FX decimals: only the forward flow was tested, refunds broke. Buyer FX: small live traffic first | Currency expansion, FX decimals |
| 7 | **Make it observable** | If it fails, I should know quickly and know why | FXCenter: completeness metric and Grafana alert. AI production triage reads error logs | FXCenter, AI triage |
| 8 | **Keep it simple, right-size it** | Choose the simplest design that fits the real scale and users | EasyFin: under 50 users, so page check-out for single-user editing, not real-time collaboration. Kept Finance's Excel habit | EasyFin |

另有一条用于 AI：**remove ambiguity**（如果人觉得不清楚，AI 也会）。

### Story → principles (按故事反查：讲这个故事时，可以点哪几条原则)

编号对应上面的 8 条原则。**主原则**是讲这个故事时优先点的；**可附带**是被追问或想串联时再用。

| Story / project | 主原则 | 可附带 | 一句话（把故事挂在原则上） |
| --- | --- | --- | --- |
| **EasyFin** | 1 Fail early | 8 Right-size, 5 Backward compatible | "Validate at submission, so a wrong journal type is caught before review. And I kept Finance's Excel habit, with a simple page check-out for under 50 users." |
| **TasCreed**（无限重试） | 2 Design for failure, 5 Fix at the root | 4 Safe to re-run | "Backoff limits the damage, a kill API lets people step in. I fixed the framework so every team benefited, and fixed file names make re-runs safe." |
| **FXCenter**（本地缓存） | 3 Protect the downstream | 7 Observable, 2 Design for failure | "Lookups use a local cache, incomplete data falls back to the previous full set, no retry loop, and a metric plus alert makes it visible." |
| **Buyer FX**（受控放量） | 6 Verify with evidence | 2 Safe failure | "Couldn't test every currency pair, so small live traffic went through first, and failed tokens were not restored automatically." |
| **币种扩展**（25 个币种未配置） | 6 Verify with evidence, test every flow | 7 Observable (trace logs) | "Instead of arguing, I tested. 25 currencies were not configured, and the refund flow had the same problem." |
| **FX 小数位事故**（5 → 7） | 6 Test every flow | 5 Backward compatible, 7 Observable (alert) | "Only the payment flow was tested, so the refund pages broke. A rollback isn't a clean undo, so I think about the data first." |
| **共享函数事故** | 5 Backward compatible | 6 Verify | "Add a new function rather than change a shared one, and check who else calls it." |
| **Payment Graph**（跨团队阻塞） | 6 Understand the whole flow, verify | 8 Right-size | "I studied the code myself and gave a concrete plan, so the other team's change was easy to make." |
| **AI skills + MCP** | Remove ambiguity | 4 Safe to re-run (repeatable flow), 7 Observable (triage) | "If a person finds an instruction unclear, the AI will too. Narrow, repeatable tasks became stable." |

### Principle → stories (反过来：被问某条原则时选哪个故事)

| Principle | First choice | Second choice |
| --- | --- | --- |
| 1 Fail early | EasyFin | Currency expansion (test early) |
| 2 Design for failure | TasCreed | Buyer FX (safe failure) |
| 3 Protect the downstream | FXCenter | none |
| 4 Safe to re-run | TasCreed | none |
| 5 Fix at the root, backward compatible | TasCreed (root) / Shared function (compatibility) | FX decimals (rollback and data) |
| 6 Verify with evidence, test every flow | Currency expansion | FX decimals, Buyer FX |
| 7 Make it observable | FXCenter alert | AI triage |
| 8 Keep it simple, right-size | EasyFin | none |

**同一轮面试里的使用建议：**
- 一个完整故事只讲一次。被问到别的原则时，换一个故事，或者用上表 "可附带" 里的一句话带过。
- 最适合串联的一对：**FX 小数位事故 → 币种扩展**（同一条原则 6，先犯错、后改进）。
- 原则 3、4、8 目前各只有一个故事，被问到时直接用它，不要硬凑第二个。

### Spoken versions (30 秒，面试官问某一条时用)

**1. Fail early**
> I like to catch errors at the earliest point. In EasyFin, a Finance user once wrote SFX 1.3 instead of BFX 1.3, a seller journal type instead of a buyer one, and it took several rounds to find. So I validate at submission, and an invalid journal type is caught right away, before anyone reviews it.

**2. Design for failure**
> I assume things will fail, so I limit the damage and keep a way for people to step in. In our task framework, some tasks retried endlessly. I added exponential backoff so retries slow down, and a kill API so people can always stop a task. In Buyer FX, I also chose to be safe: failed transactions did not restore the token automatically, a person checked first.

**3. Protect the downstream**
> In FXCenter, Search and View Item had about ten times the traffic of Checkout, so the client reads rates from a local cache, and only calls the server to preload. If the data was incomplete, it used the previous day's full set instead of retrying, and raised an alert, so we could fix it quickly.

**4. Safe to re-run**
> Our framework runs tasks at least once, so a task may run twice. Each report shard writes to a fixed file name, so a re-run just overwrites the old file, and we don't get duplicates.

**5. Fix at the root, stay backward compatible**
> When I saw endless retries in a shared framework, I fixed the framework itself, so every team benefited. And when I change shared code, I check who else uses it, and prefer adding a new function over changing the old one. I learned that from a release where I changed a shared function and set off an alert in another application.

**6. Verify with evidence, test every flow**
> When I'm not sure, I test, and I test every flow. In the currency expansion project, the PM thought E2E testing wasn't needed. I ran tests, and 25 currencies turned out not to be configured. Before that, in an FX decimal change, we had only tested the payment flow, and the refund pages broke. So I now test refund and protection flows too.

**7. Make it observable**
> In FXCenter, an incomplete rate set produced a metric and a Grafana alert. I want to know quickly when something is wrong, and why. I also use an AI workflow that reads production error logs and does a first analysis.

**8. Keep it simple, right-size it**
> EasyFin had fewer than 50 eligible users and no need for real-time collaboration, so I used a simple page check-out, with a timeout, for single-user editing. I also kept Finance's Excel habit, so there was almost nothing new to learn.

### If asked: "What coding principles do you follow?" (60 秒总答案)

选 3 条，别全说：

> Three that I follow. First, **fail early**: in EasyFin, I validate at submission, so a wrong journal type is caught before anyone reviews it. Second, **design for failure**: in our task framework, I added backoff for endless retries and a kill API, so people can always step in. Third, **verify with evidence and test every flow**: in a currency expansion project, tests showed 25 currencies were not configured, and in an earlier FX decimal change, only testing the payment flow let a refund problem through. I'm happy to go deeper into any of these.

### Notes

- "Bounded retries"：TasCreed 里我确认做的是**指数退避**，没有确认加了最大次数。不要说成 "max attempts"。
- 不要说成"我设计了整个 FXCenter 平台"，我负责的是客户端缓存、完整性检查、fallback、监控。
- 不要把安全性和性能数字说出来，除非有真实数据。

---

## 3. Technical difficulties / pushing a project to success

### Which story for which question

| Question | Story | Spoken angle |
| --- | --- | --- |
| "A hard technical problem you solved" | Buyer FX controlled rollout | Couldn't create test orders for every currency pair, so I controlled the risk with small live traffic |
| | EasyFin: keep Excel and the database in sync | Users edit a local copy, so I only allow marked rows to change and compare the rest |
| | FXCenter caching | Protect the server from 10x traffic |
| "How did you get a project done despite blockers" | Payment Graph | Blocked by another team's overloaded engineer, I did the investigation for him |
| "A risk nobody else saw" | Currency expansion | Tested, found 25 currencies not configured |
| "A project you led" | Buyer FX (in-house) | Tech lead from design to production |

### Template for a technical difficulty (90 秒)

> **Challenge** (what was hard) → **Why it was hard** (constraint) → **My approach** (the key idea, one or two steps) → **Result** → **What I'd do again or differently.**

### Example answer: Buyer FX controlled rollout

> In the in-house Buyer FX project, the hard part was testing. We couldn't create test orders for every currency pair in production, and we had no budget for a third party. So I controlled the risk. We used a token-based traffic controller to let a small amount of live traffic go through the new path. For busy pairs, the token was deducted before the transaction. If a transaction failed, the token was not restored automatically, a person checked it first, because we preferred under-testing to sending too much traffic. In the end, we completed the smoke testing across all the currency pairs, 33 payment currencies by 7 listing currencies, in about a week.

(细节和 Pessimistic / Optimistic 模式的边界见 `buyer-fx-story.md`，不要说成 exactly-once。)

### How I push a project to success (一句话原则)

- Understand the whole flow before starting (Buyer FX: case studies, shared merchant accounts).
- Surface risk early, with evidence (currency expansion).
- Remove other people's blockers (Payment Graph).
- Communicate early about delays (priorities answer).

---

## 4. Production incident / bug

### Framework for any incident (60–90 秒)

> **What happened** (impact) → **How we found out** → **Stop the bleeding** (rollback, mitigation) → **Root cause** → **Fix** → **Prevention** (what changed).

### My main example: FX decimals from 5 to 7

> We increased the FX rate decimals from 5 to 7 for VND and COP, because the precision was too low to apply the markup. It looked simple, so I changed FXCenter quickly. But we only tested the payment flow. The buyer protection pages limited the rate to 5 decimals, so they couldn't open, and buyers couldn't start a refund. The protection team alerted us. We rolled back. But the rates were already stored with 7 decimals, so we set the read side back to 5. When the protection team fixed their side, we turned 7 back on, and hard-coded the affected time range to keep the rates buyers saw consistent. Since then I test refund and protection flows for any decimal change, and I did that in the currency expansion project.

- 要点：认错（"I made the change"），回滚不是干净的撤销，预防措施是具体的（测所有流程），并连接到币种扩展。
- 追问（读写配置为何分开、为何截断不取整、多少买家）见 `interview-cards.md` 的卡片，里面有 `[待补]`。

### Smaller example: shared function

> A release where I changed a shared function set off an alert in another application. It was a false alert, but my change caused it. I rolled back, wrote the RCA, and released again with a new function and the old one untouched. Now I check who else uses shared code before I change it.

### How I debug a production issue (when asked about the process)

> First, stop the impact, for example with a rollback. Then I use the alerts, logs and traces to find the scope and cause. After the fix, I write an RCA and add a prevention step. I also use an AI workflow that reads production error logs and does a first analysis, assigns known patterns to the owner, and emails the team for new ones.

---

## 5. Unit test, CI/CD

完整问答见 `testing-ci-observability-qa.md`。这里只留要点：

> We aim for at least 85% unit test coverage, but I see it as a baseline. I focus on money logic, state changes and failure paths, and I mock external dependencies, for example with JUnit and Mockito. CI runs on Jenkins: build, unit tests, spotless and PMD. Our deployment pipeline goes through a feature pool with automation tests, then staging, then production, and a failing test stops it from moving on. For payments, I make sure the E2E tests cover all the flows, including refunds and protection flows, not just the main one.

- 面试官追问框架细节时：承认 "我们团队用 X"，回到 "我怎么决定测什么"。
- 不说内部工具名（ECD、Velocity），说阶段。

---

## 6. Bridging map: if they ask X, use Y

| If they ask... | Story | Principle to name |
| --- | --- | --- |
| "What coding principles do you follow?" | The 3-principle answer above | Fail early, design for failure, verify with evidence |
| "Tell me about a hard technical problem" | Buyer FX controlled rollout | Verify with evidence, safe failure |
| "How do you make a system reliable?" | TasCreed retries, FXCenter fallback | Design for failure, protect the downstream |
| "How do you handle duplicates or re-runs?" | TasCreed fixed file names | Idempotent |
| "A production incident you handled" | FX decimals | Test every flow, a rollback isn't a clean undo |
| "A mistake you made" | Shared function (small) or FX decimals | Backward compatibility, test every flow |
| "How do you test?" | Testing summary above + currency expansion | Layers, failure paths, E2E for all flows |
| "How do you release safely?" | Pipeline stages + traffic controller | Small steps, stop on failure, rollback |
| "How do you monitor?" | FXCenter metric and alert, triage workflow | Observable |
| "How do you keep a design simple?" | EasyFin check-out for under 50 users | Right-size |
| "Disagreement" | Currency expansion | Evidence over opinion |
| "Blocked by another team" | Payment Graph | Lower their cost |

---

## 7. A sample conversation (how it flows)

**Interviewer:** What coding principles do you follow?
**Me:** Three: fail early, design for failure, and verify with evidence. [one example each, 20 seconds]. Happy to go deeper into any of them.

**Interviewer:** Tell me more about the last one.
**Me:** In a currency expansion project, the PM thought E2E testing wasn't needed... [60-second version]. That came from an earlier incident, where an FX decimal change broke the refund pages, because we had only tested the payment flow.

**Interviewer:** What happened in that incident?
**Me:** [60–90 second incident answer]

**Interviewer:** How do you test and release today?
**Me:** [testing and CI/CD summary]

注意这条线怎么自然串起来：原则 → 币种扩展 → FX 小数位事故 → 测试和发布流程。你不用背脚本，只要知道故事之间的连接点。

---

## 8. Pitfalls

- 不要背整段。面试官在互动，打断时顺着他的问题走。
- 不要一次讲完所有例子。讲一个，留钩子。
- 不要说内部术语：FMM、RMM、CKO、ECD、Velocity。说 "payment flow"、"refund flow"、"Checkout.com"、"deployment pipeline"。
- 不要编数字。没有测量的收益，就说 "clearly less repetitive work, little rework"。
- 数字要统一：币种扩展的 "42" 是国家/地区还是币种 `[待补：确认后全局统一]`。
- 承认不知道也可以："I haven't built that myself, but I understand the idea."

## Checklist before the interview

- [ ] 能不看稿讲出 3 条原则的总答案（60 秒）。
- [ ] 能分别讲 Buyer FX、币种扩展、FX 小数位事故的 90 秒版。
- [ ] 能用 30 秒说完测试和 CI/CD 流程。
- [ ] 能把 FX 小数位事故和币种扩展连起来讲。
- [ ] Why Checkout.com 的第三个理由，Aspire / Excel / Unite 的原文。
- [ ] 统一 "42" 的说法。
