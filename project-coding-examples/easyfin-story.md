# EasyFin: Interview Story and Follow-up Questions

本文用于英文面试练习，基于本人提供的项目经历整理。重点是保留 Finance 用户的 Excel 工作习惯，同时把需求变更变成可校验、可审批、可追踪的流程。未确认的并发控制、审批保护和生产发布机制明确标注，不作为既成事实。

## 1. 故事主线

为什么做工具 → 五个功能：UI、History、Recon、Validation、Integration → 改动更清楚、沟通更少。

| 要素 | 已知内容 |
| --- | --- |
| 原流程 | Finance 本地修改 Excel，上传 Google Drive，邮件通知工程师；工程师理解需求并实施，遇到 typo 或歧义需多轮沟通 |
| 目标 | 规范需求提交与变更记录，提前发现错误，保留用户熟悉的 Excel 工作方式 |
| 个人责任 | 独立开发整个 EasyFin 工具；另一个工程审批系统不属于本故事的个人实现范围 |
| 功能 1：UI | 提供易用界面，让 Finance 提交、review 和 approve changes，保留 Excel 导入方式 |
| 功能 2：History | 当前表与 history 表保留逐行变化，便于追踪修改人和改动，支持审计 |
| 功能 3：Recon | 通过 global ID 与变更标记识别本次修改；未修改部分必须与 DB 完全一致，帮助发现误改或旧文件 |
| 功能 4：Validation | 按用户业务需求定制校验，检查 typo、无效 account 和不存在的 journal type 等错误 |
| 功能 5：Integration | 与独立工程审批系统集成，通过 batch job 自动传递批准的数据，由下游按规则生成 SQL；工程师审批后应用到 staging |
| 追问素材：编辑控制 | 页面 checkout 获取独占编辑权，锁定期间其他用户不能编辑；这个取舍不放在五个功能的主线中 |
| 已确认结果 | Finance 更容易了解变更过程；typo 和无效数据导致的沟通开销明显减少，没有提供量化指标 |

范围边界：已确认流程到 staging。不要描述成 Finance 审批后直接修改生产配置，也不声称独立工程审批系统由本人开发。

### 1.1 口述提示卡：看关键词，不背全文

先讲为什么开发工具，再按 1–5 介绍功能，最后讲结果。每个功能用自己的话讲 1–2 句；锁、表结构和具体校验实现留给追问。

| 顺序 | English cues | 想起什么 |
| --- | --- | --- |
| 原来怎么做 | Excel → Google Drive → email → back and forth | 手动传文件、通知工程师，错误和歧义导致多轮沟通 |
| 1. UI | easy to use → submit → review → approve | 通过一个易用界面管理变更，仍可以用 Excel |
| 2. History | each row → who changed what → audit | 保留逐行变化，供用户追踪和审计 |
| 3. Recon | unchanged rows → match DB → old file / accidental changes | 本次未修改部分必须与 DB 一致，发现文件或误改 |
| 4. Validation | business checks → typos → account / journal type | 按业务需求检查明显错误，减少沟通 |
| 5. Integration | approved changes → engineer tool → SQL → review → staging | 需求传递和 SQL 生成自动化，但保留工程师审核 |
| 结果 | clear changes → fewer questions | 改动更清楚，反复沟通减少 |

五个功能只记：UI → History → Recon → Validation → Integration。

卡住时用："Before EasyFin..." / "The tool had five main features." / "First..." / "Second..." / "Third..." / "Fourth..." / "Finally..." / "In the end..."。

## 2. 30-second Overview

> I built EasyFin to help Finance manage changes without so many emails and follow-up questions. It had five main features: an easy-to-use UI, row-level history, checks against the database, business-specific validation, and integration with the engineering tool. Finance could still work in Excel, but changes were easier to check, approve, and track.

## 3. Main Story: Simple Spoken English

> I built a tool called EasyFin to help Finance manage requirement changes.
>
> Before EasyFin, Finance edited Excel files, uploaded them to Google Drive, and emailed engineers. Engineers then had to work out what had changed. Typos and unclear requirements often led to several rounds of questions.
>
> I wanted to make this easier while letting Finance keep using Excel. The tool had five main features.
>
> 1. **An easy-to-use UI.** Users could upload their Excel files and submit changes in one place. Another Finance colleague could review and approve them there.
> 2. **Row-level change history.** We kept a history for each row. Users could see what had changed and who had changed it. This also helped with audits.
> 3. **Checks against the database.** Users marked the rows they wanted to change. Every other row had to match the database. If it didn't, we showed the differences. This helped catch accidental changes or an old requirement file.
> 4. **Business-specific validation.** We added checks based on Finance's needs. They caught typos and invalid data, such as an account or journal type that didn't exist. Users could fix these errors before review.
> 5. **Integration with the engineering tool.** After Finance approval, a batch job sent the changes to a separate system. That system turned them into SQL using business rules. This automated the handoff and SQL generation. Engineers still reviewed and approved the SQL before it ran in staging.
>
> In the end, Finance could track changes more easily. We also spent much less time going back and forth about typos and invalid data.

讲述提示：

- I 用于本人开发和作出的设计判断；we 用于系统流程及团队协作。
- 主线按五个功能讲，使用 First、Second、Third、Fourth、Finally；每点说明功能与用途，不先展开技术实现。
- "需求语言转为工程语言的自动化"指数据传递与 SQL 生成，不等于自动审批或直接生产发布。
- 取舍不只讲"多人协作难做"，而是解释复杂度与实际需求不匹配。人数少是背景，不单独证明并发编辑永远没有价值。
- Excel normalization 指系统可识别的标准格式，不等同于数据库范式设计。
- 不声称彻底消除了人工沟通，或有未提供的效率提升百分比。
- AI 当时尚未像现在普及不是这个故事的中心；重点仍是需求理解和工程判断。

## 4. Technical Deep Dive

### 4.1 Excel Recon：不是发现不同就覆盖 DB

```text
规范化 Excel
  -> 通过 global ID 匹配 DB 中的记录
  -> 有 Insert / Update / Delete 标记：识别为本次提交的变更
  -> 未标记记录：必须与 DB 一致
  -> 展示不一致、缺失或多出的数据，要求用户检查
  -> 通过校验的提交进入 EasyFin 等待审批
```

仅用于说明的简化例子，非真实财务数据：

| Excel 行 | 解释 |
| --- | --- |
| ID 101，标记 Update，修改某字段 | 显式提出修改；仍需通过引用等业务校验 |
| ID 102，未标记，与 DB 一致 | 未提出修改的基准数据 |
| ID 103，未标记，但与 DB 不一致 | Recon 差异，可能来自旧文件或未标记的误改，需用户检查 |

> A difference wasn't automatically an update. Users had to mark the rows they wanted to change. Everything else had to match the database.

边界：这个规则能识别未标记数据的差异，但不自动证明标记 Update 的旧数据不会覆盖其他人的新修改。版本号、提交基线、标记行的冲突检查，以及校验与写入是否原子，均需另补实际机制。

### 4.2 两层审批与 staging

```text
Finance 编辑 Excel
  -> 导入 EasyFin
  -> 校验与 recon
  -> 提交，等待另一位 Finance 同事审核批准
  -> Batch job 传递 approved data
  -> 独立工程审批系统按业务规则生成 SQL
  -> Engineer review / approval
  -> Apply to staging
```

Finance 审批和工程审批的对象与责任不同：前者审核需求变更，后者审核生成的工程变更。当前未确认 Finance 与 engineer 身份隔离的具体强制机制，也没有生产发布步骤，不自行加入。

### 4.3 Exclusive Checkout 与历史记录

- 单人编辑是五个功能之外的设计取舍，作为追问素材：不到 50 位 eligible users 不需要多人实时协作，因此选择页面 checkout。
- 页面 checkout 用于获取独占编辑权；其他用户在页面锁定期间不能编辑。
- 当前数据表与 history 表用于记录每条数据的变化。
- History 能支持追踪，不代表已经实现任意版本回滚、不可篡改审计或完整审批关联。
- 页面锁不自动等于分布式锁；锁存储、后端写入保护、释放、过期与故障恢复方式尚未确认。

## 5. Follow-up Questions

### User Needs and Scope

**1. "Why did you keep Excel instead of replacing it with a web editor?"**

> Finance was used to Excel. I wanted to add checks and approval, and make changes easier to track, without making them learn a new way to work.

回答重点：保留用户习惯，同时把控制点放在导入和变更流程。不要声称做过未提供的用户研究。

**2. "Why did you choose exclusive editing rather than real-time collaboration?"**

> We had a small user group, and they didn't need to edit the same page at the same time. Letting one person edit at a time was much simpler and met their needs.

回答重点：不到 50 位有提交资格的用户是背景；结合实际工作方式解释取舍，不只说实现太难。

**3. "What did you personally build?"**

回答重点：本人独立开发 EasyFin，包括导入校验、变更管理、审批流程、页面编辑控制与历史记录。独立工程审批系统负责生成和审批 SQL，不归为本人实现。

待补：如果追问需求访谈、测试、部署或维护，按实际职责回答，不仅凭"开发整个工具"推断全部环节均由一人完成。

### Validation and Concurrency

**4. "How did you know which changes the user meant to make?"**

> Each record had a global ID. Users marked rows as insert, update, or delete. Rows without a mark had to match the database. If they didn't, we showed the differences to the user.

回答重点：显式意图加基准一致性检查，不对所有差异自动更新。

**5. "How did you handle an outdated spreadsheet?"**

回答重点：旧文件可能导致未标记记录与 DB 不一致，系统展示 recon 差异，用户检查并修正。

准确表达：

> The check helped us spot old data or changes without a mark. That alone doesn't mean every version conflict was handled.

待补：标记行的冲突检测，以及有没有版本号或基准快照。不把现有规则扩大为完整防覆盖保证。

**6. "What kinds of validation did the tool perform?"**

> We added checks based on Finance's needs. For example, the tool could flag typos or a reference to an account or journal type that didn't exist.

回答重点：规范格式、typo 等输入问题，以及业务前置条件，例如引用的 account、journal type 是否存在于当前配置。Recon 检查未改数据的一致性，business validation 检查提交内容的合法性，是不同功能。

待补：具体字段、类型、枚举和定制校验的配置方式。不要声称系统自动纠正所有 typo、理解所有财务语义，或用户能在线编写校验代码。

**7. "What happened if a user checked out a page and disconnected?"**

待确认：锁释放、过期、人工解锁或其他恢复方式。不能仅根据 checkout 按钮就声称存在 lease、TTL 或自动续期。

后续还可能问：

> How did the backend make sure only one person could edit?

需说明实际后端保护；前端禁用按钮本身不足以证明并发写入安全。

**8. "Could data change between reconciliation and submission?"**

待确认：Recon 与写入的事务边界、锁保护范围和版本校验。页面独占编辑是否覆盖其他页面、batch job 或外部修改，也需要实际证据。

这是面试准备边界，不代表已确认系统存在缺陷；不要临时编造事务锁或 CAS 实现。

### Approval, Integration, and Audit

**9. "Did Finance approval directly update accounting configurations?"**

> No. We automated the handoff and SQL generation, not the approval. Engineers still reviewed the SQL before it ran in staging.

回答重点：保留两层 review，明确已知终点是 staging，不称为 Finance 批准后直接生产生效。

**10. "What happened if the batch job failed or delivered the same change twice?"**

待确认：传递状态、重试、幂等标识、失败告警，以及下游如何去重。不要因为存在 batch job 就假设系统有 exactly-once 或分布式事务。

**11. "What did the audit history contain?"**

回答重点：当前表保留最新数据，history 表记录逐条数据的变化；用户可以了解变更及修改人。

待补：具体是否包含前后值、时间、审批记录关联，以及历史写入是否与当前数据更新原子。不要声称可以回滚或日志不可篡改。

**12. "Could someone approve their own changes?"**

回答重点：已描述流程要求另一位 Finance 同事 review / approve。

待确认：系统是否按提交人身份强制禁止自审，以及权限如何控制。流程要求不等于已经核实技术强制机制。

### Outcomes and Engineering Principles

**13. "What changed after launch?"**

> Finance could see what had changed and who had changed it. We also had fewer rounds of questions about typos or accounts that didn't exist.

回答重点：可追踪性和沟通开销的改善是已知结果，没有可引用的具体效率数字。

待补：一个实际用户反馈或被提前发现的错误，若没有就不虚构案例。

**14. "What engineering principle guided your design?"**

> I tried to keep the design as simple as possible while still meeting the users' needs. The tool checked inputs and recorded changes, rather than leaving all of that to emails and conversations.

回答重点：简单性不等于省略正确性控制；减少不必要的多人协作复杂度，同时保留校验、审批和历史。

若被问下一次会改什么，需结合真实经历；锁恢复、版本冲突与 batch 幂等可以作为设计讨论方向，不说成已经确认的失败或改进。

## 6. Useful Expressions

| English | 中文与使用边界 |
| --- | --- |
| Finance requirement management | 财务需求管理 |
| An easy-to-use UI | 易用界面，让用户提交、审核和批准变更 |
| Row-level change history | 每一行的变化历史，用于追踪与审计 |
| Business-specific validation | 根据用户业务需求定制的校验 |
| Automate the handoff and SQL generation | 自动传递需求并生成 SQL，不等于自动审批 |
| Standardized spreadsheet format | 规范化的表格格式 |
| Proposed change | 提议的变更，尚未审批生效 |
| Explicit change labels | 显式 Insert / Update / Delete 标记 |
| Reconciliation / reconcile against the database | 对照数据库核对，识别差异 |
| Invalid reference | 无效引用，例如不存在的 account |
| Exclusive editing access | 独占编辑权限 |
| Check out a page | 获取页面编辑权，与 Git checkout 无关 |
| Record-level change history | 逐条记录的变更历史 |
| Review and approval | 审核与批准 |
| Apply changes to staging | 应用到预生产测试环境，不是生产环境 |
| Back-and-forth communication | 多轮反复沟通 |
| Communication overhead | 沟通开销 |
| Scope trade-off | 在实际需求与实现复杂度之间作取舍 |

## 7. 下一轮需要补齐的事实

- 页面锁的存储、后端执行、释放与异常恢复机制。
- 标记行是否做基线或版本冲突检查，recon 与写入的原子性。
- 草稿、提交、Finance 审批后，当前表与 history 表分别何时更新。
- Batch job 的失败恢复、重试、去重与状态跟踪方式。
- History 的实际字段，以及与审批和数据写入的关系。
- 防自审和权限控制的实际实现。
- Staging 之后的生产发布边界，若不属于本人职责则明确说明。
- 一个真实用户反馈、校验案例或个人反思。

练习顺序：先说为什么开发工具 → 按 UI、History、Recon、Validation、Integration 讲五个功能 → 结果。再练问题 2、4、5、6、9、13、14，按追问展开锁、校验和审批实现；不逐字背稿。

## 8. Coding principles with examples（第一轮用）

以下例子均来自已确认的设计，没有新增事实。

| 原则 | 我的例子 |
| --- | --- |
| Fail early：尽早发现错误 | Finance 把 BFX 1.3 误写成 SFX 1.3；提交时按业务规则校验 journal type、account 等，错误在 review 之前就被拦住 |
| 对照事实检查：不信输入 | Recon：只有标记的行允许改，其余行必须与 DB 一致，帮助发现误改或使用旧文件 |
| 可追踪、可审计 | 当前表加 history 表，保留逐行变化，能看到谁改了什么 |
| 简单优先，按规模取舍 | 不到 50 位 eligible users，不需要多人实时协作，所以用页面 checkout 单人编辑 |
| 保留用户习惯，自动化交接但留人工审核 | 仍可导入 Excel；批准的数据自动生成 SQL，但工程师审批后才应用到 staging |

> One principle I follow is to fail early. In EasyFin, a Finance user once wrote SFX 1.3 instead of BFX 1.3, a seller journal type instead of a buyer one, and it took several rounds to find. So I validate at submission, and the tool catches an invalid journal type before anyone reviews it. I also check that every row that wasn't marked as changed matches the database, which catches accidental edits or an old file.

注意：规模取舍的依据是不到 50 位 eligible users；History 不等于任意版本回滚或不可篡改审计（见 4.3）。
