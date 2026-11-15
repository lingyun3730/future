# AI Engineering Workflow: Interview Story and Follow-up Questions

本文用于英文面试练习，基于本人提供的真实经历整理。主线：把重复、可复制的开发任务做成"专用 skill + 本地 MCP"的自动化流程，人只在最后 review。未提供的数据（节省时间、功能数量等）标为 [待补]，不要编造。

## 1. 故事主线

| 要素 | 已知内容 |
| --- | --- |
| 背景 | Data Gateway team，很多 feature 是处理 reports 和 webhooks：需求简单、模式固定、重复度高 |
| 问题 | 每个 feature 都要经历：读 Jira → 写代码 → 单元测试 → 在 feature environment 做集成测试 → 贴结果 → 提 PR，大量重复劳动 |
| 做法 | 开发 skill 配合 MCP，覆盖从读 Jira ticket 到提 PR 的整条流程 |
| Skill（都在 ai-plugins repo） | 1) 开发 skill：coding style、unit test 要求；2) 集成测试 skill：配合 MCP 使用；另有规范的 PR description 模板 |
| MCP | 本地 MCP，连接 feature cloud 环境：触发 CI build manifest、部署代码到 feature pool、发送 payload 测试、收集结果 |
| 人的角色 | 整个流程不需要人工参与，只在最后一步 review 结果 |
| 迭代 | 经过几个 feature 的迭代后趋于稳定，基本不需要返工 |
| 具体例子 | 要 consume 某 processor 的某个 report events：把 incoming event spec 给 AI，让它按 Jira 里要求的输出 event spec 做 mapping。已有 processor 的 mapping 规则大多相似，AI 学习现有 processor 的 mapping，除非有特殊需求，否则保持一致 |
| 可信度 | integration test 结果如实写进 PR description，由人最后 review |
| 局限 | 需求模糊时 AI 会出错，例如让它设计 dynamic pricing 的 buyer FX 需求，它没有深入理解整个系统，方案偏通用，看似正确但无法落地 |
| 个人原则 | 极力消除给 AI 指令的不确定性：任何人觉得不确定的地方，对 AI 也要讲清楚，否则结果就是一团浆糊 |

### 1.1 口述提示卡

| 顺序 | English cues | 想起什么 |
| --- | --- | --- |
| 背景 | reports and webhooks → simple, repeatable | 需求简单、重复 |
| 做法 | skill + local MCP → Jira to PR | 一整套自动化流程 |
| Skill | coding style / unit test / integration test / PR template | 规范写进 skill |
| MCP | feature env → build → deploy → send payload → collect results | 集成测试自动化 |
| 例子 | learn existing processor mappings | 学习已有 mapping，保持一致 |
| 人的角色 | review at the end | 最后只 review |
| 局限 | vague requirements → generic design | 需求模糊时 AI 会出错 |
| 原则 | remove ambiguity | 指令必须清楚 |

## 2. 30-second Overview

> In my team, many features were about handling reports and webhooks, and they were simple and repetitive. So I built skills and a local MCP server that cover the whole flow: reading the Jira ticket, writing code and tests, running integration tests in a feature environment, and opening a PR. I only review the result at the end. After a few features, it became stable and needed little rework.

## 3. Main Story: Simple Spoken English

> In my team, many features were about processing reports and webhooks. These tasks were simple and very similar to each other, but each one still needed the same steps: read the Jira ticket, write the code, write unit tests, run integration tests in a feature environment, and open a PR.
>
> So I automated this flow with skills and an MCP server.
>
> The skills define the rules. One skill covers the coding style and unit test requirements. Another covers integration tests. There's also a PR description template. All of them live in our ai-plugins repo.
>
> The MCP server runs locally and connects to our feature environment. It can trigger a CI build, deploy the code to a feature pool, send test payloads, and collect the results.
>
> For example, when we need to consume a new report's events from a processor, I give the AI the incoming event spec. It matches it to the output event spec in the Jira ticket. Most existing processors use similar mapping rules, so the AI learns from them and follows the same pattern, unless there's a special requirement.
>
> It writes the code in our coding style, runs the tests, and puts the real integration test results in the PR description. I only review at the end.
>
> After a few features, the flow became stable, and I rarely had to redo anything.

讲述提示：

- I 用于本人设计 skill 和 MCP 的判断；"AI" 做执行，"I" 负责定规则和最终 review。
- 重点讲**为什么适合**：任务专一、重复、规则固定，所以 skill 能写得很具体，几乎不出幻觉。
- 不说"完全不需要人"，说"整个流程不需要人工参与，最后 review 结果"。
- 不声称有未提供的量化数字；可口述时说"明显提效"，具体数字 [待补]。

## 4. 面试官可能追问

**Q: Why skills and MCP, and how are they different?**
A skill is instructions: it tells the AI how to do a type of task, such as our coding style or test requirements. MCP is a tool connection: it lets the AI do things in the outside world, like deploy to a feature environment and send requests. The skill decides what to do and how. The MCP does the actions. Used together, the AI can do the whole flow.

**Q: What if the AI makes a mistake or hallucinates?**
For this type of task, it rarely happens. The skill is very narrow and describes exactly what to do, and the AI follows existing processors as examples. Also, integration tests run in a real feature environment, and the results are written in the PR as they are. Then I review the PR as I would for any code. Mistakes happen mostly when requirements are vague.

**Q: Where does AI not work well? Give an example.**
When the requirement is vague or needs deep system knowledge. For example, I asked it to design a dynamic pricing requirement for buyer FX. It didn't dive into the whole system, so its design was general and looked right, but it couldn't fit into our current system. For this kind of work, I use AI to explore options, but I make the design decision myself.

**Q: How did you make it stable?**
By iterating. After each feature, I looked at what needed rework and put it into the skill: coding style, test requirements, PR template. After a few features, it rarely needed changes. [待补: 具体改过哪些规则。]

**Q: What's your principle for working with AI?**
Remove uncertainty from the instructions. If a person would find something unclear, the AI will, too, so I spell it out. Otherwise the result is a mess.

**Q: How do you make sure the code is good and safe?**
The skill requires our coding style and unit tests. Integration tests run in the feature environment. The PR needs the normal CI checks, such as spotless and PMD, and I review it before it's merged. [待补: 确认 skill 里是否包含 spotless / PMD 要求。] Sensitive data isn't sent to external systems. [待补: 确认公司的具体策略。]

**Q: How do you measure the benefit?**
[待补: 每个 feature 原来多久、现在多久；用这套流程完成了多少个 feature。没有数据就说"明显减少了重复劳动，返工很少"，不要编数字。]

**Q: Can other teammates use it?**
The skills are in the ai-plugins repo, so others can use and improve them. [待补: 是否有其他人在用、谁在维护。]

**Q: What would you improve next?**
[待补: 你的真实想法。可能的方向：覆盖更多类型的重复任务；把失败的案例回写进 skill。]

## 5. 使用提醒

- 主线讲"自动化重复任务，规则讲清楚"，不要讲成"AI 能做任何事"。
- 局限性（需求模糊）要主动承认，这会让回答更可信。
- 工具名称：面试时说 Claude Code 和 GitHub Copilot 均可；不确定哪个 skill / MCP 在哪个工具里运行时，说"I use skills and MCP with these AI tools"。
