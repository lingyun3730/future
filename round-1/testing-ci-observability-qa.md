# Testing, CI/CD, Testability, Observability: Interview Q&A

目标：让面试官觉得你**懂原理、有实践**，但不要陷在具体框架的细节里。
原则：
- 先讲"为什么、怎么分层"，再点一两个工具名（JUnit、Mockito），不展开 API。
- 被追问细节时，说"我了解思路 / 我们团队用的是 X"，再把话题带回原则和我的真实例子。
- `[待补]` = 需要我补真实情况，不要编。

---

## Part 1: Unit tests

### 1. How do you approach unit testing?

**要点：** 测行为，不测实现。每个测试小而快，不依赖外部系统。外部依赖用 mock 替换。覆盖正常路径、边界和失败路径。

> I test behavior, not implementation. Each unit test is small and fast, and doesn't depend on external systems, so I mock the dependencies, for example with JUnit and Mockito in Java. I cover the normal path, the edge cases, and the failure paths, like timeouts and invalid input. In payments, the failure paths matter most.

### 2. What do you mock, and what don't you?

**要点：** mock 外部依赖（数据库、银行、网络、时间）。不 mock 被测代码自己的逻辑。mock 太多会让测试只是在重复实现，反而没用。

> I mock things outside the unit: database, network, other services, the clock. I don't mock the logic I'm testing. If I need many mocks to test one thing, it usually means the design has too many dependencies, so I simplify it.

### 3. How do you decide how much to test? Do you chase coverage?

**要点：** 我们的要求是单元测试覆盖率不低于 85%。但覆盖率只是底线，不是目标：重点测关键逻辑、金额计算、状态转换、错误处理。高覆盖率也可能没测到真正重要的东西。

> Our standard is at least 85% unit test coverage. But I see that as a baseline, not the goal. I focus on the logic that matters: money calculations, state changes and error handling. High coverage can still miss the important cases, so I think about what could go wrong first.

### 4. How do you test a function that calls an external service?

**要点：** 单元测试里 mock 它，模拟成功、失败、超时。再用集成测试在真实环境里验证一次。

> In a unit test I mock the service and simulate success, failure and timeout. Then I verify it again in an integration test in a real environment, because a mock only checks my assumptions about the service.

### 5. What makes code easy to test?

**要点：** 依赖通过构造函数传入（依赖注入）、函数职责单一、尽量少的隐藏状态和全局变量、时间和随机数可以替换。

> Small functions with one job, dependencies passed in rather than created inside, and no hidden global state. Time and randomness should be replaceable. If code is hard to test, it's often a sign the design needs work.

---

## Part 2: Integration and E2E tests

### 6. How do your tests layer?

**要点：** 单元测试：多、快、便宜。集成测试：在真实或接近真实的环境里，验证组件之间配合。E2E：少而关键，覆盖完整用户流程（包括退款这类非主流程）。

> Many fast unit tests at the bottom, then integration tests in a real feature environment, and a smaller number of end-to-end tests for the key flows. For payments, E2E should cover every flow, not just the main one: payment, refund and the protection flows.

**我的例子：**
- Data Gateway：集成测试在 feature environment 里跑，发 payload，收集结果，真实结果写进 PR。
- 币种扩展：E2E 发现 25 个币种没配置，退款流程也有同样问题。
- FX 小数位事故：只测了正向流程，漏了退款和保护页面。

### 7. How do you test when you can't test everything?

**要点：** 用风险控制：小流量先上线、受控放量、人工确认再放大。

> I control the risk. In Buyer FX, we couldn't create test orders for every currency pair, so we sent a small amount of live traffic through the new path first, and a person checked failures before we allowed more.

### 8. How do you test failure paths, like timeouts and retries?

**要点：** 在 staging 或测试里主动制造故障：超时、返回错误、数据不完整。确认系统的反应符合设计（告警、回退、不重复处理）。

> I trigger them on purpose in a test environment: timeouts, errors, incomplete data. Then I check that the system does what we designed: it alerts, it falls back, and it doesn't process twice. In FXCenter, we tested the incomplete-data path in staging, and it didn't happen in production.

---

## Part 3: CI/CD

### 9. What does your CI/CD pipeline do?

**要点：** CI 用 Jenkins：编译、单元测试（覆盖率不低于 85%）、spotless 和 PMD 检查、构建。发布走内部的 CD 流水线（ECD），包含：CI build → 部署到 feature pool → 自动化测试 → 部署 staging → 部署 production。任何一步失败（测试或自动化测试），就**无法**继续部署到 staging 和 production。

面试时：内部工具名（ECD、Velocity）可以不说，或说一次 "our internal deployment pipeline"。外部面试官不认识它们，说阶段更清楚。

> We use Jenkins for CI: it builds the code, runs the unit tests with at least 85% coverage, and the static checks, spotless for formatting and PMD for code quality. Our deployment pipeline then runs in stages: CI build, deploy to a feature pool, automation tests, deploy to staging, and then production. If a test or an automation test fails, the change can't go to staging or production. So a broken change is blocked before it reaches users.

### 10. How do you release safely?

**要点：** 小步发布、先在测试环境验证、能快速回滚、发布后看监控和告警。数据变化要提前想好，回滚不一定能完全撤销。

**我们的流程：** 变更要依次通过 feature pool 的自动化测试、staging，才能上 production，任何一步失败都不能往下走。

> Every change goes through the same stages: a feature pool with automation tests, then staging, then production, and a failure stops it from moving on. After release, I watch the metrics and alerts, and I make sure I can roll back quickly. And I think about data: a rollback is not always a clean undo. In the FX decimal change, we rolled back, but the rates were already stored with 7 decimals, so we had to handle the data too.

### 11. What would make a build or merge fail?

**要点：** 编译错误、测试失败、覆盖率低于 85%、格式检查（spotless）失败、静态检查（PMD）出现新问题。自动化测试失败也会阻止部署到 staging 和 production。

> A compile error, a failing test, coverage under 85%, a formatting failure, or a new static-analysis issue. A failing automation test also blocks the deployment to staging and production, so any problem has to be fixed first.

### 12. How do you deal with a flaky test?

**要点：** 不要反复重跑直到通过。先找原因：共享状态、时间、顺序依赖、异步等待。修好，或者暂时隔离并记录，不能留着让大家习惯性忽略。

> I don't just re-run it until it passes. I look for the cause: shared state, time, test order or async waiting. I fix it, or isolate it and track it, because ignoring a flaky test teaches the team to ignore failures.

---

## Part 4: Observability

### 13. What is observability, and how is it different from monitoring?

**要点：** 监控：知道"出问题了"（预设的指标和告警）。可观测性：能回答"为什么"（日志、指标、trace 一起，能查预先没想到的问题）。

> Monitoring tells me something is wrong, based on metrics and alerts I defined in advance. Observability lets me find out why, using logs, metrics and traces together, even for problems I didn't expect.

### 14. What do you log, and what do you not log?

**要点：** 记录关键事件、ID（请求 ID、交易 ID）、错误原因。不记录敏感数据（卡号、密钥、个人信息）。日志要有足够的上下文，能把一次请求串起来。

> I log key events with IDs, like the request or transaction ID, and the error reason, so I can follow one request across services. I never log sensitive data like card numbers or secrets.

### 15. What metrics and alerts do you set up?

**要点：** 成功率、错误率、延迟、队列积压、关键业务指标。告警要能行动，太多告警会被忽略。

**我的例子：** FXCenter 的 completeness 检查产生 metric，Grafana 告警，人工修复后重启客户端。

> I watch the success and error rate, latency, and queue backlog, plus a few business metrics. Alerts should be actionable, so people don't ignore them. In FXCenter, an incomplete rate set produced a metric and a Grafana alert, so we could fix it quickly.

### 16. How do you debug a production issue?

**要点：** 先止血（回滚、降级），再排查。看告警、日志、trace，找出范围和根因，修复，写 RCA，加预防措施。

> First stop the impact, for example by rolling back. Then I look at the alerts, logs and traces to find the scope and the cause. After the fix, I write an RCA and add a prevention step. In the FX decimal incident, we rolled back first, and then dealt with the data and the tests.

**也可以提：** 我用 MCP 让 AI 读线上 error log，做初步分析并分派，已知 pattern 分配给对应的人，新的发邮件给组里（production triage）。

---

## Part 5: If the interviewer digs

| If they ask | Safe answer |
| --- | --- |
| "Which mocking library, how does `when().thenReturn()` work?" | "We use Mockito. I use it to replace a dependency and set its response. I'm happy to write it, but the key is what I choose to mock." |
| "What's your coverage threshold?" | "At least 85%. I treat that as a baseline, and I care more about covering the critical logic." |
| "Which CI tool?" | "Jenkins for CI, and an internal pipeline for deployment: feature pool, automation tests, staging, production." |
| "Differences between mock, stub, spy?" | "A stub returns fixed data, a mock also lets me verify calls, and a spy wraps a real object. I mostly use mocks and stubs." |
| "Testing pyramid?" | "Many unit tests, fewer integration tests, and a few E2E tests, because the higher the layer, the slower and more expensive." |
| "Test-driven development?" | "I don't always write tests first, but I design for testability, and for bug fixes I write a failing test first." `[待补：只在真实时说]` |

## Tips

- 面试官问框架细节时，别紧张：承认 "我们团队用 X，我了解它的思路"，再回到 "我怎么决定测什么"。
- 每个回答尽量带一个我的真实例子（币种扩展 E2E、FX 小数位事故、FXCenter 告警）。
- 不确定的数字（覆盖率、工具名）不要编。
