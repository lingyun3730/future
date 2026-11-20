# AI Follow-up Q&A for Interviews

用途：面试官聊到 AI 时可能追问的基础原理和真实使用。不会很深，所以每题只写：一句中文要点 + 简单英文回答。
我的真实经历见 `../round-1/ai-workflow-story.md`（skills + MCP 自动化 Data Gateway 任务、production triage）。
`[待补]` = 需要我自己补真实情况，不要编。

---

## Part 1: Basic principles / 基本原理

### 1. How does an LLM work?

**要点：** 大语言模型是预测"下一个 token"的模型。用海量文本训练，学到语言和知识的统计规律。它不是在"查数据库"，所以会一本正经地编造（幻觉）。

> An LLM predicts the next token, one at a time, based on the text so far. It is trained on a huge amount of text, so it learns patterns of language and knowledge. It doesn't look facts up in a database, so it can sound confident and still be wrong. That is why I give it clear context and always check the result.

### 2. What is a token and a context window?

**要点：** token 是模型处理文本的最小单位（一个词或词的一部分）。context window 是一次能"看到"的 token 上限，包括输入和输出。超出就看不到，或者要截断。

> A token is a small piece of text, like a word or part of a word. The context window is how many tokens the model can see at once, including my input and its output. If I give too much, it may lose important details, so I keep the context focused.

### 3. What is a hallucination, and how do you reduce it?

**要点：** 模型编造看似合理但不真实的内容。减少办法：给明确的上下文和例子、让它基于给定资料回答（RAG）、缩小任务范围、用测试和 review 验证。

> A hallucination is when the model makes up something that sounds right but isn't. I reduce it by giving clear context and examples, keeping the task narrow, and verifying the result with tests and review. In my Data Gateway flow, the tasks are narrow and the AI follows existing processors as examples, and the integration test results are real, so problems are rare.

### 4. What does temperature do?

**要点：** 控制随机性。低 = 更确定、更稳定；高 = 更多样、更有创造性。写代码、自动化任务用低；头脑风暴可以高。

> Temperature controls randomness. Low is more stable and predictable, which I want for code and automation. Higher is more varied, which can help with brainstorming.

### 5. Prompt, RAG, or fine-tuning: how do you choose?

**要点：** 先从最简单的开始。
- **Prompt（提示词）：** 最便宜最快，先试这个。
- **RAG：** 模型缺少你的私有或最新知识时，先检索相关资料，再放进上下文让它回答。知识常变也适合。
- **Fine-tuning：** 要改变模型的风格或行为，且有足够数据时才用，成本最高。

> I start with a good prompt. If the model lacks our private or fresh knowledge, I use RAG: retrieve the relevant documents and put them in the context. Fine-tuning is the last option, when I need to change the model's behavior or style and have enough data. It costs the most.

### 6. What is RAG, and what are embeddings?

**要点：** RAG = 检索增强生成：先按问题检索相关资料，再交给模型生成答案。Embedding = 把文本变成向量，意思相近的文本向量也相近，所以可以做语义搜索。向量放在向量数据库里。

> RAG means retrieval-augmented generation. First we search for relevant documents, then we give them to the model with the question. Embeddings turn text into vectors, so similar meanings are close to each other, and we can search by meaning. They are stored in a vector database.

*我自己没有做过 RAG 系统 [待补，若没有就如实说：我了解原理，主要实践是 skills + MCP]。*

### 6.1 What is context engineering?

**要点：** 决定每次给模型看什么：指令、例子、相关资料、工具结果。给得准、给得少而精，比写一个"魔法提示词"更重要。skill 就是把可重复的上下文写成文件。

> It is deciding what the model sees each time: instructions, examples, relevant documents, tool results. Giving the right context, and not too much, matters more than a clever prompt. Skills are how I make that context repeatable.

---

## Part 2: Agents, tools, MCP, skills

### 7. Agent vs workflow

**要点：** Workflow = 步骤由代码预先写死；Agent = 由 LLM 自己决定下一步和用哪些工具。能用 workflow 就用 workflow，因为更便宜、更可预测。

> A workflow follows a path written in code. An agent lets the LLM decide its own steps and tools. Workflows are cheaper and more predictable, so I use them when the steps are known. I use an agent when the task is open-ended and I can't list the steps in advance.

**我的例子：** Data Gateway 的 feature 自动化步骤固定，skill 规定每一步，更接近 workflow；production triage 的"分析日志"需要 AI 判断，带一点 agent 的成分。

### 8. What is tool calling (function calling)?

**要点：** 模型本身只能输出文本。Tool calling 让它输出"我要调用哪个工具、参数是什么"，由外部程序真正执行，再把结果交还给模型。模型是"决定"，程序是"执行"。

> The model can only produce text. With tool calling, it outputs which tool to call and with what arguments, our program runs it, and the result goes back to the model. The model decides, the program acts.

### 9. What is MCP, and how is it different from a skill?

**要点：** MCP（Model Context Protocol）是让 AI 连接外部工具和数据的统一协议。Skill 是一份写给 AI 的说明，告诉它某类任务该怎么做。Skill = 做什么、怎么做；MCP = 能做的动作。

> MCP is a standard protocol that lets an AI connect to outside tools and data. A skill is instructions that tell the AI how to do a type of task. The skill says what and how; the MCP provides the actions. In my flow, the skill describes the test steps, and the MCP deploys to the feature environment and sends payloads.

### 10. How do you give the AI the right context in real work?

**要点：** 写成可复用的 skill（编码规范、测试要求、PR 模板）、给现有的例子（已有 processor 的 mapping）、把任务缩小、指令里消除歧义。

> I put the rules into skills: coding style, test requirements, the PR template. I let it learn from existing examples, like existing processors' mappings. And I remove ambiguity: if a person would find it unclear, the AI will too.

---

## Part 3: Real usage / 真实使用

### 11. How do you use AI tools in your daily work?

**要点：** 用 Claude Code 和 GitHub Copilot。不只是写代码（vibe coding），而是把重复任务做成 skill + MCP 的流程；也用来分析线上日志（production triage）。设计决策我自己做。

> I use Claude Code and GitHub Copilot. Beyond coding, I built skills and a local MCP server to automate repetitive Data Gateway tasks, from reading the Jira ticket to opening a PR. I also use them for production triage: the AI reads production error logs and does a first analysis. I make the design decisions myself.

### 12. Where does AI not work well? Give an example.

**要点：** 需求模糊、需要深入理解整个系统的时候。例子：让它设计 buyer FX 的 dynamic pricing 需求，它没有深入理解系统，方案偏通用，看似正确但落不了地。这类工作我用 AI 探索方案，但自己做决定。

> When the requirement is vague or needs deep knowledge of the whole system. For example, I asked it to design a dynamic pricing requirement for Buyer FX. The design was general and looked right, but it didn't fit our system. For work like that, I use AI to explore options, but I decide myself.

### 13. How do you make sure AI-generated code is correct and safe?

**要点：** 和人写的代码一样：规范写进 skill、单元测试、集成测试在真实环境跑、结果如实写进 PR、CI 检查、人最后 review。

> I treat it like any other code. The skill requires our coding style and unit tests. Integration tests run in a real feature environment, and the real results go into the PR description. The normal CI checks run, and I review it before merge.

*[待补：skill 里是否包含 spotless / PMD；公司对数据的具体政策。]*

### 14. What are the security and privacy risks?

**要点：**
- 不把敏感数据（密钥、客户数据、内部机密）发到外部系统。
- **Prompt injection：** 输入里藏着恶意指令（例如网页、日志内容），诱使 AI 做不该做的事。所以对 AI 读到的内容不能全信，工具权限要最小化，危险操作要人确认。
- 给 MCP / agent 的权限要最小，尽量只读。

> I don't send sensitive data like secrets or customer data to external systems. There is also prompt injection: text in the input, like in a log or a web page, can try to trick the AI into doing something else. So I give tools the least permission they need, and I keep a human review for risky actions. My production triage only reads logs, and it only assigns or sends email.

### 15. How do you measure the benefit?

**要点：** 没有精确数字就如实说：明显减少重复劳动，返工很少。不要编数字。

> I haven't measured exact time saved. What I see is much less repetitive work and little rework after the skills became stable, and teammates adopted it on their own, which is a good sign. `[待补：如果有每个 feature 的耗时对比，写在这里。]`

### 16. What would you improve next?

**要点：** 把失败的案例回写进 skill；覆盖更多类型的重复任务；给 production triage 增加更多已知 pattern。`[待补：用自己的真实想法。]`

---

## Part 4: Quick phrases

| Term | One line |
| --- | --- |
| Token | Small piece of text the model reads and writes |
| Context window | How much text the model can see at once |
| Hallucination | Confident but wrong or made-up answer |
| Temperature | Randomness of the output |
| Prompt | The instruction and context we give the model |
| RAG | Search documents first, then let the model answer from them |
| Embedding | Text as a vector, so similar meaning is close |
| Fine-tuning | Train the model further on our data |
| Tool calling | Model chooses a tool and arguments, our code runs it |
| MCP | Standard protocol to connect AI to tools and data |
| Skill | Written instructions for a type of task |
| Workflow | Fixed steps, written in code |
| Agent | LLM decides its own steps and tools |
| Prompt injection | Hidden instructions in the input that trick the AI |

## Tips

- 面试官的追问通常不深：用一句话讲清楚原理，再接一个我的真实例子。
- 不懂的就说"I know the idea, but I haven't built it myself"，不要硬撑。
- 我的真实实践是 skills + MCP + production triage，不是 RAG 或 fine-tuning，不要说成做过。
