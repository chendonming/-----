# Claude Agent SDK 多 agent 编排官方怎么做？

> **一句话总结**：**在 Claude Agent SDK 里做多 agent 编排，官方给的答案只有两条路——① subagents（用 `agents` 参数声明式定义子代理，Claude 在运行时按 `description` 自动委派，天然支持并行/后台/嵌套/通信）；② Workflow 工具（把编排写进一段 JS 脚本交给运行时执行，用于"几十到几百个 agent"的规模化编排）。** 注意：当前官方文档里**不存在** `Agent` 类、`run_agent()`、`Orchestrator`、`sendMessage()` 这一套命令式 API——凡是让你 `new Agent()` 或 `orchestrator.run()` 的写法，都来自旧版 Claude Code SDK 或别的产品，别被带偏。
>
> 本文基于 Claude Agent SDK 官方文档 `Agent SDK overview`、`Subagents in the SDK`、`TypeScript / Python SDK reference`、`Migrate to Claude Agent SDK`、`Orchestrate subagents at scale with dynamic workflows`，以及官方博客 `A harness for every task: Dynamic workflows in Claude Code` 整理，文末附参考来源。

先说清楚一个定位问题，避免后面被名字绕晕：**"Claude Agent SDK"是一个独立的库产品**——官方原话是 "Build production AI agents with Claude Code as a library"（把 Claude Code 当库，构建生产级 AI agent）。它和 Claude Code CLI 是两个表面：CLI 的 `agent teams`、`agent view` 是终端交互界面下的功能，不直接等价于 SDK 里的编排。本文只讲 SDK 里怎么写代码。

---

## 一、先定位：SDK 的多 agent 能力在官方文档里的位置

`Agent SDK overview` 的能力清单表里，**Subagents** 和 Hooks、MCP、Sessions、Skills 并列为 SDK 的核心能力，官方一句话定位：

> "Subagents — Spawn specialized agents for focused subtasks"
> （subagents——生成专门处理聚焦子任务的专用 agent。）

包名也确认过（`Migrate to Claude Agent SDK`）：TypeScript 是 `@anthropic-ai/claude-agent-sdk`，Python 是 `claude-agent-sdk`；API 入口是 `query()`（流式，一次任务，两种语言都有），Python 另有 `ClaudeSDKClient` 类（连续多轮会话，TS 是函数式 API、没有这个类）。多 agent 编排就在这些入口的 `options` 里配置。

## 二、核心机制：subagents —— 声明式定义，Claude 运行时编排

`Subagents in the SDK` 文档第一句就把机制讲透了：

> "Subagents are separate agent instances that your main agent can spawn to handle focused subtasks."
> （subagent 是**独立**的 agent 实例，由你的主 agent 生成，用来处理聚焦的子任务。）

关键在"**由 Claude 决定**"：你只负责定义，什么时候派谁出去是 Claude 在运行时决定的——

> "When you define subagents, Claude determines whether to invoke them based on each subagent's `description` field. Write clear descriptions that explain when to use the subagent, and Claude automatically delegates appropriate tasks."
> （定义了 subagent 后，**Claude 根据每个 subagent 的 `description` 字段决定何时调用它**。把"什么场景该用"写进 description，Claude 就会自动委派合适的任务。）

### 定义 subagent 的三种方式

| 方式 | 说明 | 官方推荐度 |
|---|---|---|
| **代码里定义**（`agents` 参数） | 在 `query()` 的 options 里传 `agents` 字典，每个 key 是 agent 名，value 是 `AgentDefinition` | **推荐**（"recommended for SDK applications"） |
| 文件系统定义 | `.claude/agents/` 下的 markdown 文件 | 备选，与 CLI 一致 |
| 内置 `general-purpose` | 不定义任何东西，Claude 也能通过 Agent 工具临时派发通用 subagent | 免配置 |

### AgentDefinition 配置字段

`Subagents in the SDK` 文档给出的完整字段表（Python 里字段名保持 camelCase，`disallowedTools`、`mcpServers` 等不转 snake_case）：

| 字段 | 必填 | 作用 |
|---|---|---|
| `description` | ✅ | 自然语言描述"什么时候该用这个 agent"，Claude 凭它决定是否委派 |
| `prompt` | ✅ | 该 agent 的系统提示词，定义它的角色与行为 |
| `tools` | 否 | 允许的工具白名单；省略则继承全部 |
| `disallowedTools` | 否 | 从工具集中移除哪些（支持 `mcp__server__*` 等模式） |
| `model` | 否 | 模型覆盖，如 `'fable'`/`'opus'`/`'sonnet'`/`'haiku'`/`'inherit'` |
| `skills` | 否 | 预加载进该 agent 上下文的 skill 名列表 |
| `memory` | 否 | 记忆来源：`user`/`project`/`local` |
| `mcpServers` | 否 | 该 agent 可用的 MCP server |
| `initialPrompt` | 否 | 作为主线程 agent 时自动提交的第一条用户消息（当 subagent 被调用时忽略） |
| `maxTurns` | 否 | 最大 agentic 轮数 |
| `background` | 否 | 是否作为**非阻塞后台任务**运行 |
| `effort` | 否 | 推理努力等级 `low`~`max` 或数字 |
| `permissionMode` | 否 | 该 agent 内的工具执行权限模式 |

### 官方示例：两个并行 subagent（代码审查 + 跑测试）

这是官方文档的代码审查示例，是最典型的多 agent 编排形态——一个只读审查员、一个能跑命令的测试员，同时定义：

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review the authentication module for security issues",
  options: {
    // 必须把 Agent 放进 allowedTools，subagent 调用才会自动放行、不弹权限
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      "code-reviewer": {
        description: "Expert code review specialist. Use for quality, security, and maintainability reviews.",
        prompt: "You are a code review specialist... Be thorough but concise in your feedback.",
        tools: ["Read", "Grep", "Glob"],   // 只读：不能改文件
        model: "sonnet",
      },
      "test-runner": {
        description: "Runs and analyzes test suites. Use for test execution and coverage analysis.",
        prompt: "You are a test execution specialist...",
        tools: ["Bash", "Read", "Grep"],   // 能跑命令
      },
    },
  },
})) {
  if ("result" in message) console.log(message.result);
}
```

Python 对应写法（`claude_agent_sdk`）：

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="Review the authentication module for security issues",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Glob", "Agent"],
        agents={
            "code-reviewer": AgentDefinition(
                description="Expert code review specialist. Use for quality, security, and maintainability reviews.",
                prompt="You are a code review specialist...",
                tools=["Read", "Grep", "Glob"],
                model="sonnet",
            ),
            "test-runner": AgentDefinition(
                description="Runs and analyzes test suites.",
                prompt="You are a test execution specialist...",
                tools=["Bash", "Read", "Grep"],
            ),
        },
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

两个易错点，官方都专门提了：

- **必须把 `Agent` 加进 `allowedTools`**，否则 subagent 调用会落入 `canUseTool` 回调、在 `dontAsk` 模式下直接被拒。
- **`description` 决定委派**。想让 Claude"保证用某个 subagent"，就在 prompt 里点名："Use the code-reviewer agent to check the authentication module"（这是文档里明确写的显式调用方式）。

### 动态生成 agent 定义（工厂模式）

subagent 定义不必是写死的字典——官方在 "Dynamic agent configuration" 一节展示了用**工厂函数**按运行条件生成 `AgentDefinition` 的写法（比如"严格评审用 opus、常规评审用 sonnet"；创建时机在 `query()` 时，所以每次请求可以用不同设置）：

```python
def create_security_agent(security_level: str) -> AgentDefinition:
    is_strict = security_level == "strict"
    return AgentDefinition(
        description="Security code reviewer",
        prompt=f"You are a {'strict' if is_strict else 'balanced'} security reviewer...",
        tools=["Read", "Grep", "Glob"],
        model="opus" if is_strict else "sonnet",   # 高利害审查用更强模型
    )

# 调用处：agents={"security-reviewer": create_security_agent("strict")}
```

动态编排不止"运行时派谁出去"，连 **agent 的定义本身也可以按输入生成**——同一套编排逻辑对不同任务配不同子代理，这是官方示例之外的常见扩展。

## 三、编排细节：并行、后台、嵌套、通信、续跑

多 agent 编排不止"定义多个"，官方把运行时行为也讲得很细。

### 1. 并行：多个 subagent 并发跑

官方给并行下了一个非常直白的定义（这是 SDK 里多 agent 最核心的价值）：

> "Multiple subagents can run concurrently, so independent subtasks finish in the time of the slowest one rather than the sum of all of them."
> （多个 subagent 可以**并发**运行，所以相互独立的子任务，完成时间是**最慢那个**的用时，而不是所有任务用时之和。）

配套的上下文隔离是它可行的前提：

> "Each subagent runs in its own fresh conversation. Intermediate tool calls and results stay inside the subagent; only its final message returns to the parent."
> （每个 subagent 跑在**全新的会话**里。中间的工具调用和结果都留在 subagent 内部，**只有最终消息返回给父 agent**。）

### 2. 后台运行：`background` / `run_in_background`

`AgentDefinition.background` 字段官方解释：

> "Run this agent as a non-blocking background task when invoked."
> （被调用时，作为**非阻塞后台任务**运行。）

而自 Claude Code v2.1.198 起，subagent 本身就默认后台跑：

> "Subagents run in the background by default. An Agent tool call that omits the `run_in_background` input launches a background subagent, and Claude sets `run_in_background: false` when it needs the result before continuing."
> （subagent 默认**在后台**运行。Agent 工具调用不传 `run_in_background` 时启动的是后台 subagent；当 Claude 需要结果才能继续时，才把 `run_in_background` 设为 `false` 以前台方式运行。）

配套的控制面在 `Query` 对象上：`stopTask(taskId)`（Python 是 `ClaudeSDKClient.stop_task(task_id)`）停掉指定后台任务；`agentProgressSummaries: true` 会让后台/前台 subagent 的一行进度摘要通过 `task_progress` 事件转发给你；`forwardSubagentText: true` 则把 subagent 的文本/思考块也当作带 `parent_tool_use_id` 的 assistant/user 消息转发出来，方便消费方渲染嵌套转录（默认只转发 `tool_use`/`tool_result` 块）。还有环境变量 `CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS`（默认 600000ms）做后台 subagent 的卡死看门狗。

### 3. 嵌套：subagent 还能再生 subagent

> "By default, subagents can spawn subagents of their own, up to three layers below the main conversation."
> （默认情况下，subagent 可以**再生成自己的 subagent**，最多到主对话以下三层。）

改层级上限用环境变量 `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`（设 `1` 即关掉嵌套）。这在编排"树状拆解"任务时很有用。

### 4. 通信：`SendMessage` 工具 + 命名 agent 列表

subagent 之间可以直接发消息，但前提是它持有 `SendMessage` 工具：

> "A subagent that has the `SendMessage` tool starts with a list of the other named agents running in the session, so it knows which names it can send messages to."
> （持有 `SendMessage` 工具的 subagent，开局就收到一份**当前会话里其他命名 agent 的列表**，因此它知道可以向哪些名字发消息。）

这是 SDK 里唯一的 agent 间点对点通信通道。除此之外，父 agent 与 subagent 之间的"通信"全靠 Agent 工具传 prompt、以及 subagent 返回的最终消息。

### 5. 续跑（resume）：拿回 agentId 接着干

subagent 完成后，Agent 工具结果里带一段 `agentId: <id>` 文本。官方给的三步续跑流程：**①** 从消息里捕获 `session_id`；**②** 从 Agent 工具结果里解析 `agentId`；**③** 用 `resume: sessionId` 再发起一次 `query()`，并在 prompt 里带上 agentId。续跑会保留该 subagent 的完整历史（工具调用、结果、推理）。注意官方提醒：`Explore` 和 `Plan` 是 one-shot、不返回 `agentId`，要续跑得用自定义 subagent 或 `general-purpose`。

另外，官方建议通过 `parent_tool_use_id` 字段识别"这条消息来自哪个 subagent 的上下文"，配合 Agent 工具的 `tool_use`（名字是 `Agent`，老版本叫 `Task`）就能重构出完整的 agent 嵌套树。

## 四、规模化编排：Workflow 工具 —— 把编排写进脚本

subagent 有规模上限。官方在 `Subagents in the SDK` 的 "Scale up with dynamic workflows" 一节给了明确的"升级信号"：

> "Subagents work well for a few delegated tasks per turn. For runs that coordinate dozens to hundreds of agents, use the `Workflow` tool, which moves the orchestration into a script the runtime executes outside the conversation context."
> （subagent 适合**每轮几个委派任务**。要协调**几十到几百个** agent 的运行，就用 `Workflow` 工具——它把编排移进一段**在对话上下文之外**执行的脚本。）

> "The `Workflow` tool is available in the TypeScript Agent SDK v0.3.149 and later. Include `Workflow` in `allowedTools` to auto-approve workflow runs."
> （`Workflow` 工具在 TypeScript Agent SDK **v0.3.149 及以后**可用。把 `Workflow` 加进 `allowedTools` 就能自动放行 workflow 运行。）

### 判断标准：谁握着计划？

`Orchestrate subagents at scale with dynamic workflows` 文档用一张表区分了 subagents 和 workflows，核心判据是"**谁来决定下一步跑什么**"：

| | Subagents | Workflows |
|---|---|---|
| 是什么 | 一个 Claude 生成的工人 | 一段运行时执行的脚本 |
| 谁决定下一步 | Claude，逐轮决定 | 脚本 |
| 中间结果放哪 | Claude 的上下文窗口 | 脚本变量 |
| 可重复的是什么 | 工人的定义 | **编排本身** |
| 规模 | "A few delegated tasks per turn"（每轮几个） | "Dozens to hundreds of agents per run"（每次几十到几百个） |

原文说得更直白：

> "With subagents, skills, and agent teams, Claude is the orchestrator: it decides turn by turn what to spawn or assign next, and every result lands in a context window. A workflow script holds the loop, the branching, and the intermediate results itself, so Claude's context holds only the final answer."
> （用 subagent、skills、agent teams 时，**Claude 是编排者**：它逐轮决定生成/分配什么，每个结果都落进上下文窗口。而 workflow 脚本**自己持有循环、分支和中间结果**，Claude 的上下文只装最终答案。）

### 脚本长什么样

workflow 就是一段纯 JavaScript 脚本（顶层 `await`），官方给了一个最小骨架：

```javascript
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'], properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

核心函数就几个：`agent()` 生成一个 subagent；`pipeline()` 对列表里每一项各跑一遍；`Workflow` 工具参考（TS reference）里还有 `parallel()`（有屏障的并发）和 `phase()`（进度分组）。"中间结果留在脚本变量里"意味着**循环、分支、结果去重、交叉验证都可以用代码写死**，比让 Claude 在上下文里逐轮编排可靠得多。

### 运行时限制（官方给的数字）

| 约束 | 数值 |
|---|---|
| 并发上限 | **最多 16 个**并发 agent（CPU 核少的机器更少） |
| 单次运行总量 | **最多 1,000 个** agent |
| 中间态 | 无用户输入打断；脚本本身不能直接碰文件系统/shell |

成本上官方也提示过：workflow 会显著多用 token，建议先跑小切片验证，再上全量。

除 SDK 的 `Workflow` 工具外，CLI 侧生成 workflow 的入口是让 Claude 直接写脚本：prompt 里带 `ultracode` 关键词（或直接说 "use a workflow"），或 `/effort ultracode` 让会话里每个实质任务自动走 workflow。跑完的脚本可存成 `/命令` 复用（`.claude/workflows/` 项目级 / `~/.claude/workflows/` 个人级，也能进插件分发），脚本里用全局 `args` 接收每次调用的输入。运行时还有几个新护栏值得知道：**可恢复**——中途停止后已完成的 agent 返回缓存结果，未完成及其后启动的都会重跑；**Large workflow 警告**——计划超过 25 个 agent 或预计 token 超 150 万时，进度行提示去 `/workflows` 停止；**尺寸指南**——`small`(<5) / `medium`(<15) / `large`(<50) 告诉 Claude 大概写几个 agent，默认 `medium`，只作建议不是硬上限。

## 五、官方博客的六种编排模式（值得直接抄）

官方博客 `A harness for every task`（Claude Code 团队自述如何用动态工作流编排 subagent）给出了六个可组合的模式，这是"多 agent 编排到底怎么排"最实用的部分：

| 模式 | 官方原话（节选） | 一句话 |
|---|---|---|
| **Classify-and-act**（先分类再行动） | "classifier agent routes to different agents/behavior based on task type" | 分类 agent 当路由，按任务类型派给不同 agent / 不同行为 |
| **Fan-out-and-synthesize**（扇出再综合） | "Split up a task into many smaller steps, run an agent on each step and then synthesize those results" | 把任务拆成很多小步，每步一个 agent，最后综合；综合步"等所有扇出 agent，再合并它们的结构化输出" |
| **Adversarial verification**（对抗式验证） | "run a separate spawned agent to adversarially verify its output against a rubric" | 另起一个 agent，按 rubric 对抗式复查前一个 agent 的输出 |
| **Generate-and-filter**（生成再过滤） | "filter them by a rubric or by verification, dedupe duplicates" | 先生成一批候选，再按 rubric/验证过滤、去重 |
| **Tournament**（锦标赛） | "Spawn N agents that each attempt the same task using different approaches" | N 个 agent 用不同思路做同一件事，评委两两比出胜者 |
| **Loop until done**（循环到完） | "loop spawning agents until a stop condition is met... instead of a fixed number of passes" | 循环生成 agent 直到满足停止条件，而不是固定轮数 |

博客还点破了这些模式要解决的单上下文三大失败模式：**agentic laziness**（干到一半就宣称完成）、**self-preferential bias**（自己验证自己，偏爱自己的结果）、**goal drift**（多轮/压缩后偏离原始目标）。workflow 让每个 subagent 有独立上下文和聚焦目标，正是为了对抗这三者。

博客最后的告诫也值得抄进需求里：

> "dynamic workflows often use more tokens and are best suited for complex, high value tasks."
> （动态工作流通常更费 token，最适合复杂、高价值的任务。）

> "For regular coding tasks, try and ask yourself: does it really need more compute? For example, most traditional coding tasks do not need a panel of 5 reviewers."
> （对常规编码任务，先问自己：真的需要更多算力吗？比如大多数传统编码任务并不需要 5 个评审组成的评审团。）

## 六、重要提醒：当前官方 API 里"没有"什么

为写这篇笔记，我专门核验了 `TypeScript / Python SDK reference` 两个参考页（2026-08-04），结论是当前文档化的 API **不存在**以下符号：

| 常被提起的 API | 当前官方文档里 | 实际对应的官方机制 |
|---|---|---|
| `new Agent()` / `Agent` 类 | ❌ 没有 | `query()` + `agents` 参数声明式定义 subagent |
| `run_agent()` | ❌ 没有 | `background: true` / `run_in_background` 后台任务，`stopTask()` 停止 |
| `Orchestrator` 类 | ❌ 没有 | 编排决策在 Claude 手里（subagents）或脚本手里（Workflow 工具） |
| `sendMessage()` | ❌ 没有 | subagent 持有 `SendMessage` 工具 + 命名 agent 列表 |

这些符号多来自旧版 Claude Code SDK（`@anthropic-ai/claude-code`，后改名并入 Agent SDK，包名 `@anthropic-ai/claude-agent-sdk`）或网上教程对"编排"的想象。`Migrate to Claude Agent SDK` 明确讲了这次改名，官方 API 的编排姿势就是本文二至四节那套：**声明式 subagents + Workflow 工具**。写作/答题时如果看到 `run_agent()`、`Orchestrator`，请回到官方文档核对。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 抓取官方 Markdown 原文 `code.claude.com/docs/en/agent-sdk/*.md` 核对获取）：

- **Agent SDK overview** — https://code.claude.com/docs/en/agent-sdk/overview
  （SDK 定位 "Build production AI agents with Claude Code as a library"、能力清单里 Subagents 的定位、与 CLI / Client SDK / Managed Agents 的区分）
- **Subagents in the SDK** — https://code.claude.com/docs/en/agent-sdk/subagents
  （本文主干：subagents 定义与三种方式、AgentDefinition 完整字段表、并行/后台/嵌套/SendMessage/resume、Agent 工具与 `parent_tool_use_id` 检测、`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`、Workflow 工具的升级信号）
- **Agent SDK reference — TypeScript** — https://code.claude.com/docs/en/agent-sdk/typescript
  （`query()` 与 `Query` 对象、`agents`/`agent` 选项、`run_in_background`、`stopTask()`、`agentProgressSummaries`、`forwardSubagentText`；核对确认页面无 `Agent` 类 / `run_agent()` / `Orchestrator` / `sendMessage()`）
- **Agent SDK reference — Python** — https://code.claude.com/docs/en/agent-sdk/python
  （`query()` 与 `ClaudeSDKClient`、`AgentDefinition` dataclass 字段、`agents` 参数、`stop_task()`、camelCase 字段名约定；核对确认 Python 侧同样无 `Agent` 类 / `run_agent()` / `Orchestrator`）
- **Migrate to Claude Agent SDK** — https://code.claude.com/docs/en/agent-sdk/migration-guide
  （旧包 `@anthropic-ai/claude-code` → 新包 `@anthropic-ai/claude-agent-sdk` 的改名史、破坏性变更：`ClaudeAgentOptions`、system prompt 默认行为、`settingSources`）
- **Orchestrate subagents at scale with dynamic workflows** — https://code.claude.com/docs/en/workflows
  （subagents vs skills vs agent teams vs workflows 对照、"谁握着计划"、脚本骨架 `meta`/`agent()`/`pipeline()`、运行时限制：16 并发 / 1,000 总量、规模数字 "A few delegated tasks per turn" vs "Dozens to hundreds of agents per run"）
- **Claude 官方博客：A harness for every task — Dynamic workflows in Claude Code** — https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
  （"Dynamic workflows execute a javascript file with a few special functions that help spawn and coordinate subagents"、六大编排模式、三大失败模式、token 成本告诫）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 使用判据与内置 subagent 的 CLI 侧口径）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（subagent 的上下文隔离价值）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
