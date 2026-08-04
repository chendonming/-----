# Claude Code 官方文档对上下文窗口和上下文管理是怎么说的？

> **一句话总结**：官方把上下文窗口定义为一次会话的"**工作记忆**"（working memory）——装着对话历史、文件内容、命令输出、CLAUDE.md、自动记忆、已加载 skills 和系统指令；它**填得很快、满了性能会下降**，因此官方原话是 "**The context window is the most important resource to manage（上下文窗口是最需要管理的资源）**"。围绕这条根本约束，官方给的上下文管理体系分**两层**：**自动层**（接近上限时先清旧工具输出、再自动压缩对话；prompt caching 按前缀缓存复用历史；MCP 工具定义默认延迟加载）+ **手动层**（`/context` 查看占用、`/compact` 带焦点压缩、`/clear` 换任务清零、`/rewind` 部分压缩、`/btw` 旁路小问题、subagent 隔离大文件读取、CLAUDE.md 精简到 200 行内、专项指令挪进 skills）。另外，如果问题出在"窗口不够大"而不是"对话太长"，官方还给了 Fable 5 / Sonnet 5 / Opus 4.6 及之后 / Sonnet 4.6 的 **100 万 token 扩展上下文**。
>
> 本文综合 Claude Code 官方文档 `Glossary`、`How Claude Code works`、`Explore the context window`、`Best practices for Claude Code`、`Manage costs effectively`、`How Claude remembers your project`、`How Claude Code uses prompt caching`、`Extend Claude Code`、`Model configuration` 共 9 篇资料整理，文末附参考来源与获取方式。

---

## 一、官方定义：上下文窗口 = 会话的"工作记忆"

`Glossary` 给了一个最浓缩的定义：

> "The working memory for a session, holding conversation history, file contents, command outputs, CLAUDE.md, auto memory, loaded skills, and system instructions. As you work, context fills up until compaction summarizes it. Run `/context` to see what's using space."
> （上下文窗口是一次会话的**工作记忆**，装着对话历史、文件内容、命令输出、CLAUDE.md、自动记忆、已加载的 skills 和系统指令。随着你干活，上下文会被填满，直到压缩机制把它汇总。运行 `/context` 查看什么在占空间。）

`Explore the context window` 的正文第一句补充了"还包括你根本看不到的内容"：

> "Claude Code's context window holds everything Claude knows about your session: your instructions, the files it reads, its own responses, and content that never appears in your terminal."
> （Claude Code 的上下文窗口装着本次会话中 Claude 知道的一切：你的指令、它读过的文件、它自己的回复，以及**永远不会出现在你终端里**的内容。）

`How Claude Code works` 的表述把组成成分列全，并点出"满了会丢"这个关键特性：

> "Claude's context window holds your conversation history, file contents, command outputs, CLAUDE.md, auto memory, loaded skills, and system instructions. As you work, context fills up. Claude compacts automatically, but instructions from early in the conversation can get lost."
> （Claude 的上下文窗口装着对话历史、文件内容、命令输出、CLAUDE.md、自动记忆、已加载的 skills 和系统指令。随着干活，上下文会被填满。Claude 会自动压缩，但**对话早期的指令可能因此丢失**。）

### 窗口有多大

官方没有给一个"统一大小"的硬数字，而是分两档：

| 窗口 | 说明 |
|---|---|
| **默认窗口（约 20 万 token）** | `Explore the context window` 的交互式模拟以 `MAX = 200000` 演示会话如何被填满，但页面注明 **"Token counts are illustrative"（token 数是示意性的）**，实际值随你的 CLAUDE.md 大小、MCP server 数量和文件长度而变。 |
| **扩展窗口（100 万 token）** | 官方原话：**"If you need a larger window rather than a smaller conversation, Fable 5, Sonnet 5, Opus 4.6 and later, and Sonnet 4.6 support a 1 million token context window."**（如果你需要更大的窗口而不是更小的对话，Fable 5、Sonnet 5、Opus 4.6 及之后、Sonnet 4.6 支持 100 万 token 的上下文窗口。）"Compaction works the same way at the larger limit."（压缩在更大的上限下工作方式相同。） |

官方在"想要更大窗口"上的态度很明确——**先考虑把对话变小，而不是直接换大窗口**（见第六节）。

### 窗口是"整段会话"级别的

上下文窗口不是一个短暂缓存，而是 Claude Code **每次调用模型都会跟着发送的完整历史**（`prompt-caching` 文档的底层解释）：

> "Claude Code re-sends the full context: the system prompt, your project context, every prior message and tool result, and your new message."
> （Claude Code 会重新发送完整的上下文：系统提示词、项目上下文、此前的每一条消息和工具结果，以及你的新消息。）

会话之间是隔离的——`How Claude remembers your project` 开篇：

> "Each Claude Code session begins with a fresh context window."
> （每个 Claude Code 会话都从一个全新的上下文窗口开始。）

跨会话的知识只靠两条通道延续：你写的 **CLAUDE.md** 和 Claude 自己写的 **auto memory**（两者都在每次对话启动时加载）。

---

## 二、为什么上下文是"最需要管理的资源"

`Best practices for Claude Code` 把整个最佳实践体系的出发点归结为一条约束：

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."
> （**大多数最佳实践都基于同一条约束：上下文窗口填得很快，一满性能就下降。**）

官方给了两个直接原因：

1. **填充速度超出直觉**：
   > "A single debugging session or codebase exploration might generate and consume tens of thousands of tokens."
   > （**一次调试会话或代码库探索，就可能产生并消耗数万 token。**）

2. **填满后是真的会"变笨"**：
   > "When the context window is getting full, Claude may start 'forgetting' earlier instructions or making more mistakes."
   > （当上下文窗口快满时，Claude 可能开始"忘记"更早的指令，或犯更多错误。）

由此引出整份文档里最关键的一句原话：

> "**The context window is the most important resource to manage.**"
> （**上下文窗口是最需要管理的资源。**）

`How Claude Code works` 甚至把"上下文管理"直接写进了 Claude Code 的定位——它不是普通聊天外壳，而是：

> "Claude Code serves as the **agentic harness** around Claude: it provides the tools, context management, and execution environment that turn a language model into a capable coding agent."
> （Claude Code 是包在 Claude 外面的**智能体 harness**：它提供工具、**上下文管理**和执行环境，把一个语言模型变成能干的编码智能体。）

---

## 三、上下文被什么填满：启动即加载 + 干活即增长

`Explore the context window` 用一张交互式时间轴演示了一场会话。它的核心结论是——**你还没打一个字，大量内容已经进上下文了**。页面 "What the timeline shows" 一节的原文：

> "**Before you type anything**: CLAUDE.md, auto memory, MCP tool names, and skill descriptions all load into context."
> （**你还没输入任何东西之前**：CLAUDE.md、自动记忆、MCP 工具名和 skill 描述就已经全部加载进上下文了。）

### 3.1 启动自动加载项（token 数为官方示意值）

| 加载项 | 示意 token | 官方说明要点 |
|---|---|---|
| **System prompt（系统提示词）** | ~4,200 | "Always loaded first. You never see it."（永远最先加载，你永远看不到它。） |
| **Auto memory（MEMORY.md）** | ~680 | Claude 自己记的笔记；只加载 **前 200 行或前 25KB**（先到者为准）。 |
| **Environment info（环境信息）** | ~280 | 工作目录、平台、shell、OS、是否 git 仓库；git 分支/状态/近期提交作为独立块放在系统提示词末尾。 |
| **MCP tools（默认延迟加载）** | ~120 | 只列出工具名让 Claude 知道可用；完整 schema 默认延迟、经 tool search 按需取用。 |
| **Skill descriptions** | ~450 | 只加载**一行描述**；完整内容用到才加载。压缩后**不重新注入**，只有实际调用过的 skill 被保留。 |
| **`~/.claude/CLAUDE.md`（用户级）** | ~320 | 你的全局偏好，每个项目、每次对话启动都加载。 |
| **Project CLAUDE.md（项目级）** | ~1,800 | 官方最强调的文件；提示语 "Keep it under 200 lines."（控制在 200 行以内）。 |

时间轴想传递的核心信息（**转述**，非官方逐字原话）：启动时加载的项目知识——CLAUDE.md、自动记忆、MCP 工具名、skill 描述——加起来远大于你的第一条 prompt，所以"上下文大部分是项目知识，而不是你说的话"。

### 3.2 会话进行中，谁在吃掉上下文

随着 Claude 工作，主要增量来自：

- **每次文件读取**——官方明说 "File reads dominate context usage"（**文件读取主导上下文占用**），并给出对应建议：
  > "Be specific in prompts ("fix the bug in auth.ts") so Claude reads fewer files. For research-heavy tasks, use a subagent."
  > （提示词写具体一点（如"修复 auth.ts 里的 bug"），让 Claude 少读文件。研究密集型任务用 subagent。）
- 读取文件时**自动加载的路径作用域规则**（`.claude/rules/` 里带 `paths:` 的规则），终端只显示一行 "Loaded ..."；
- 每次编辑后触发的 **PostToolUse hook**——只有写入 `hookSpecificOutput.additionalContext` 的输出才进上下文；
- 你的每一条后续提问（都在同一份上下文上继续累加）。

### 3.3 各特性"常驻成本"对照：按需加载的由来

`Extend Claude Code` 的 "Understand context costs" 一节，把各种扩展特性的上下文成本汇总成表，是"为什么有的东西该按需加载"的依据：

| 特性 | 何时加载 | 加载什么 | 上下文成本 |
|---|---|---|---|
| **CLAUDE.md** | 会话启动 | 完整内容 | **每次请求都在** |
| **Skills** | 启动 + 使用时 | 启动只加载描述，用时装全文 | 低（描述每次请求都在）* |
| **MCP servers** | 会话启动 | 只列工具名，schema 按需 | 用到工具前都很低 |
| **Code intelligence** | 编辑后与按需 | 编辑后的诊断、符号定位 | 低，且能减少别处的文件读取 |
| **Subagents** | 生成时 | 全新上下文 + 指定 skills | **与主会话隔离** |
| **Hooks** | 触发时 | 不加载任何内容（外部运行） | 零，除非用 additionalContext 返回内容 |

\* 对只由你手动触发的 skill，官方建议 `disable-model-invocation: true`，让它的描述也完全不出现在上下文里，直到 `/name` 调用。

---

## 四、上下文满了怎么办：自动压缩（auto-compact）

窗口再大也会填满。官方的设计是：**满了不会让你的会话终止，而是自动压缩**。

> "Claude Code compacts automatically as you approach the limit, so a full context window doesn't end your session."
> （**接近上限时 Claude Code 会自动压缩，所以上下文窗口满了也不会结束你的会话。**）

`How Claude Code works` 描述了自动压缩的两级动作顺序：

> "Claude Code manages context automatically as you approach the limit. It clears older tool outputs first, then summarizes the conversation if needed."
> （Claude Code 在你接近上限时自动管理上下文：**先清掉较早的工具输出，必要时再把对话总结成摘要。**）

注意它的取舍：

> "Your requests and key code snippets are preserved; detailed instructions from early in the conversation may be lost."
> （你的请求和关键代码片段会被保留；**早期对话中的详细指令可能会丢。**）

所以官方紧接着给了最重要的建议——**把要长期生效的规则写进 CLAUDE.md，而不是放在对话里**：

> "Put persistent rules in CLAUDE.md rather than relying on conversation history."
> （把持久规则放进 CLAUDE.md，而不是依赖对话历史。）

**抖动保护**：如果单个文件或工具输出太大，导致每次压缩完立刻又被填满，官方明确说不会死循环：

> "If a single file or tool output is so large that context refills immediately after each summary, Claude Code stops auto-compacting after a few attempts and shows an error instead of looping."
> （如果某个文件或工具输出大到"每次压缩后立刻又满"，Claude Code 会在几次尝试后**停止自动压缩并报错**，而不是无限循环。官方称其为 thrashing error，排查见 Troubleshooting 文档。）

### 压缩之后：什么活着、什么会丢

`context-window` 文档有一张"压缩存活表"（What survives compaction），是全文信息密度最高的一张表：

| 机制 | 压缩之后 |
|---|---|
| **System prompt 与 output style** | 不变（不属于消息历史） |
| **项目根 CLAUDE.md 与无 `paths:` 的规则** | **从磁盘重新注入** |
| **Auto memory（自动记忆）** | **从磁盘重新注入** |
| **带 `paths:` 的路径作用域规则** | **丢失**，直到再次读到匹配的文件 |
| **子目录里的嵌套 CLAUDE.md** | **丢失**，直到再次读到该子目录的文件 |
| **调用过的 skill 正文** | **重新注入**，但每个 skill 上限 5,000 token、总量上限 25,000 token，最老的先被丢弃 |
| **Hooks** | 不适用（hook 是代码，不是上下文） |

两个反直觉的坑，官方特别标注：

- **Skill 描述不重载**：启动时那行 skill 描述列表，压缩后**不重新注入**——"Only skills you actually invoked get preserved."（只有你实际调用过的 skill 会被保留。）
- **Skill 被截断保头**："Truncation keeps the start of the file, so put the most important instructions near the top of `SKILL.md`."（截断保留文件开头，所以最重要的指令要放在 `SKILL.md` 顶部。）

对应的处置建议（`context-window` 文档 "What survives compaction" 一节）：

> "If a rule must persist across compaction, drop the `paths:` frontmatter or move it to the project-root CLAUDE.md."
> （如果某条规则必须在压缩后仍存在，去掉 `paths:` 前置元数据，或把它挪到项目根 CLAUDE.md。）

---

## 五、主动管理：官方给的"手动阀门"

自动压缩是兜底，但官方明确鼓励**在自动触发前主动出手**（"You can also act before the automatic pass runs"）。核心命令与习惯如下：

| 手段 | 官方定位 | 关键原话 |
|---|---|---|
| **`/context`** | 查看上下文占用、拿优化建议 | "Run `/context` to see what's using space."（运行 `/context` 查看什么在占空间。）"run `/context` for a live breakdown by category with optimization suggestions, including which CLAUDE.md and auto memory files loaded."（实时按类别分解，带优化建议，含哪些 CLAUDE.md 和记忆文件已加载。） |
| **`/clear`** | 换任务时清空上下文，从零开始 | "run `/clear` when switching to unrelated work. Old conversation crowds out the files you need next and costs tokens on every message."（切换到无关任务时运行 `/clear`。旧对话挤掉你接下来需要的文件，还在每条消息上耗 token。） |
| **`/compact <focus>`** | 手动把对话压缩成结构化摘要，**可带焦点** | "Run `/compact` with instructions, like `/compact focus on the auth bug fix`, before starting a long new task. The summary keeps what you choose instead of what the automatic pass guesses is important."（长任务前带焦点运行 `/compact`，保留你选的而不是自动压缩猜的。） |
| **`/rewind` / `Esc Esc`** | **部分压缩**：只压缩一部分对话 | "To compact only part of the conversation, use `Esc + Esc` or `/rewind`, select a message checkpoint, and choose **Summarize from here** or **Summarize up to here**."（前者压缩该点之后、保留早前完整；后者压缩早前、保留最近完整。） |
| **`/btw`** | 快速小问题不进上下文 | "For quick questions that don't need to stay in context, use `/btw`. The answer appears in a dismissible overlay and never enters conversation history."（答案出现在可关闭浮层里，**永不进对话历史**。） |
| **状态栏持续显示** | 让上下文占用常驻可见 | "configure your status line to display it continuously."（配置状态栏持续显示上下文占用。） |

### 压缩的"焦点"与自定义指令

有两种方式控制压缩时保留什么（`how-claude-code-works` 原话）：

> "To control what's preserved during compaction, add a 'Compact Instructions' section to CLAUDE.md or run `/compact` with a focus (like `/compact focus on the API changes`)."
> （要控制压缩时保留什么，在 CLAUDE.md 里加一个 "Compact Instructions" 小节，或带焦点运行 `/compact`。）

CLAUDE.md 里的写法示例（`costs` 文档）：

```markdown
# Compact instructions

When you are using compact, please focus on test output and code changes
```

`/compact` 与 `/clear` 的关键区别——`/compact` 本身是一次"读整个要压缩的对话"的大请求，而 `/clear` 只是清空：

> "When you want a fresh start instead of continuity, `/clear` costs nothing."
> （当你想要全新开始而不是延续时，`/clear` 不花 token。）

### 两个"结构性"管理手段：隔离 + 按需

- **subagent 隔离大读取**：`how-claude-code-works` 说 subagent 拥有自己全新的、与主对话完全独立的上下文；官方给出的数字对比最直观：
  > "The subagent read 6,100 tokens of files. You got a 420-token result. That's the context savings."
  > （subagent 读了 **6,100 token** 的文件，你只收到 **420 token** 的结果。**这就是上下文节省量。**）
  `Best practices` 把它的地位抬到最高："Since context is your fundamental constraint, subagents are one of the most powerful tools available."（既然上下文是你的根本约束，**subagent 就是可用工具里最有力的一个**。）（完整使用判据见 `Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`。）

  另外，**主会话的 auto memory 不会加载进 subagent**（`fork` 子代理除外——它继承父会话的系统提示词与历史；`memory` 文档原话 "The main conversation's auto memory isn't loaded into subagents"）。想给 subagent 持久记忆，需在 subagent 配置里单独开启 `memory` 字段。

- **skill / 路径规则 / MCP 按需加载**：把"不是每次都需要"的指令移出常驻上下文。`memory` 文档说 CLAUDE.md 每次会话启动就全量加载、消耗 token；`costs` 文档据此给出迁移策略：
  > "Skills load on-demand only when invoked, so moving specialized instructions into skills keeps your base context smaller. **Aim to keep CLAUDE.md under 200 lines** by including only essentials."
  > （**Skills 只在被调用时按需加载**，所以把专项指令移进 skills 能让基础上下文变小。**目标是让 CLAUDE.md 不超过 200 行**，只留必需品。）

  另外，**hook 预处理**能把海量原始输出挡在上下文之外：
  > "Instead of Claude reading a 10,000-line log file to find errors, a hook can grep for `ERROR` and return only matching lines, reducing context from tens of thousands of tokens to hundreds."
  > （与其让 Claude 读 1 万行日志找错误，不如让 hook grep 出 `ERROR` 只返回匹配行，把上下文从几万 token 降到几百 token。）

---

## 六、反方向：要更大的窗口，而不是更小的对话

官方把"把对话变小"的手段列完之后，给了另一条路——**1M token 扩展上下文**（见第一节"窗口有多大"）。`Model configuration` 的 "Extended context" 一节补充了操作细节：

| 维度 | 官方口径 |
|---|---|
| **选择方式** | 用 `[1m]` 后缀，如 `/model opus[1m]`、`/model sonnet[1m]`，或完整模型名 `/model claude-opus-4-8[1m]`；账户支持 1M 时选项会出现在 `/model` 选择器里 |
| **定价** | "The 1M context window uses standard model pricing with no premium for tokens beyond 200K."（1M 窗口按标准定价，**对超出 200K 的 token 不加价**） |
| **套餐可用性** | Max / Team / Enterprise 下 **Opus 自动升级到 1M**（无需配置）；**Sonnet 4.6 的 1M 不在自动升级内**，任何套餐都需 usage credits；Pro 套餐两者都要 usage credits；API 按量付费全量可用 |
| **Sonnet 5 天生 1M** | "On the Anthropic API, Sonnet 5 always runs with the 1M context window. There is no 200K variant, no `[1m]` suffix to select, and no usage credits required on any plan. Sessions auto-compact before the window fills, at about 967K tokens by default; set `CLAUDE_CODE_AUTO_COMPACT_WINDOW` to choose a different threshold."（Sonnet 5 永远运行在 1M 窗口，无 200K 变体、无需 `[1m]`、任何套餐都不需 credits；默认约 967K token 时自动压缩，可用环境变量调阈值） |
| **例外（退回 200K）** | 走 LLM gateway（`ANTHROPIC_BASE_URL` 指向网关）时无法验证 1M 支持，需在模型选择器里显式选 "Sonnet 5 (1M context)"；设置 `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` 则把 Sonnet 5 会话按 200K 窗口处理 |

补充：官方还区分了"**支持 1M**"与"**默认就按 1M 跑**"两件事——"支持 1M"的完整清单是 Fable 5 / Sonnet 5 / Opus 4.6 及之后 / Sonnet 4.6（需显式选 `[1m]` 或靠套餐自动升级）；而在 Anthropic API 上"**always run with the 1M window**（永远运行在 1M 窗口）"的只有 **Fable 5、Sonnet 5、Opus 4.7 及之后**这三者，无需任何选择。

注意官方把"更大窗口"和"更小对话"并列为两条路，且**默认先谈怎么把对话变小**——窗口能换，但上下文管理的功夫该花还在花，因为 "Compaction works the same way at the larger limit."（压缩在更大的上限下工作方式相同。）

---

## 七、为什么"管理上下文" = 省钱：prompt caching 视角

`How Claude Code uses prompt caching` 把上下文管理和成本直接挂钩：

> "Without caching, the API would reprocess your full history on every turn. With caching, it reuses what it already processed, bills the re-read at the cached token rate, and fully processes only what changed."
> （没有缓存，API 每一轮都要重新处理你的全部历史。有了缓存，它复用已处理过的部分，**按缓存 token 费率计费**，只完整处理变化的部分。）

缓存按**前缀精确匹配**（prefix match），并按"变更频率低→高"把请求分成三层：

| 层 | 内容 | 何时变化 |
|---|---|---|
| **System prompt** | 核心指令、工具定义、output style | 工具定义集合变化、或 Claude Code 升级 |
| **Project context** | CLAUDE.md、auto memory、无 `paths:` 规则 | 会话启动、`/clear`、`/compact` |
| **Conversation** | 你的消息、Claude 的回复、工具结果 | **每一轮都在变** |

由此得出一个实用结论——**哪些操作会让下一条消息变慢变贵**（缓存失效）：

| 会使缓存失效 | 保持缓存 |
|---|---|
| 切换模型（每个模型独立缓存） | 编辑仓库文件（下次读取才生效） |
| 切换 effort 级别 | 会话中编辑 CLAUDE.md（**不生效，重启/`/clear`/`/compact` 才应用**） |
| 开启 fast mode | 改输出风格（同上，会话内不生效） |
| MCP 服务器连接/断开（工具加载进前缀时） | 切换权限模式 |
| 启用/禁用提供 MCP server 的插件（其工具加载进前缀时） | 调用 skills 与命令 |
| 拒绝整类工具 | `/recap`、`/rewind` |
| **压缩对话（`/compact`）** | 派生子代理（子代理自己建缓存） |

> 注：`prompt-caching` 文档对"MCP / 插件是否使缓存失效"给了更精确的区分——**默认开启 tool search 的模型上，MCP 工具是延迟加载的**，连接/断开 server 或工具列表变化只是追加新内容、**不扰动已有缓存**；只有当工具**加载进前缀**（tool search 不可用/被禁用、或 server 标了 `alwaysLoad`）时，变更才会导致整段重读。插件同理：只有**提供 MCP server 且其工具加载进前缀**的插件才会使缓存失效；skills / commands / agents / hooks 等其它组件都只是追加内容，不会失效。

关于 `/compact` 的成本，官方特别指出它是"按设计使对话层失效"：

> "Compaction replaces your message history with a summary. By design, this invalidates the conversation layer…"
> （压缩用一份总结替换消息历史。**按设计，这会使对话层缓存失效。**）

但压缩请求本身利用前缀缓存，所以**会话中段的 `/compact` 成本只占上下文大小的零头**；最贵的是隔了缓存 TTL 之后恢复旧会话再压缩（`costs` 页也说 "`/compact` reads the conversation it summarizes, so compacting a large context is itself a large request"——压缩大上下文本身就是个大请求）。官方给出的习惯是：

> "Pick your model and effort level at the top of a session, then save `/compact` for natural breaks between tasks. The fewer changes you make mid-task, the higher your cache hit rate."
> （**在会话开头就定好模型和 effort 档位**，把 `/compact` 留到任务之间的自然停顿处。任务中途改动越少，缓存命中率越高。）

### 长会话为什么"没干什么却很贵"

`costs` 文档在 "Why usage climbs in a long session" 解释了那个反直觉现象：

> "Claude Code sends your full conversation with every request… a one-line question in a session that has been open all day still draws usage for the whole conversation."
> （Claude Code 每次请求都发送你的完整对话……一个挂了一整天的会话里，**问一句一行的话，也按整段对话扣用量**。）

这条现象的另一个放大器是**缓存 TTL**：`prompt-caching` 文档说明缓存条目在一段不活动后过期——订阅套餐默认 **1 小时** TTL，API key / 云厂商默认 **5 分钟**，一旦用到 usage credits 也会降为 5 分钟。所以"离开很久回来再问一句"那次请求会错过缓存、全量重读整个历史，就是"慢 + 贵"的瞬间。官方给出的缓解是：大会话隔了很久要恢复时，Claude Code 会**提供"从摘要恢复"（resume from a summary）**，让后续请求不必再扛着完整历史。

### 会话中途改 CLAUDE.md 不生效

`prompt-caching` 文档：项目根和用户级 CLAUDE.md 只在会话启动时读一次、保存在内存里——

> "Editing them mid-session does not invalidate the cache, but the edit also doesn't apply. The new content loads on the next `/clear`, `/compact`, or restart."
> （会话中途编辑它们不会使缓存失效，但改动也不生效。新内容要等下次 `/clear`、`/compact` 或重启才加载。）

例外是**子目录嵌套 CLAUDE.md** 和**带 `paths:` 的路径作用域规则**——它们到"首次读到匹配文件"时才加载，所以在加载前编辑仍然生效（`prompt-caching` 文档原话："Nested CLAUDE.md files in subdirectories and rules with `paths:` frontmatter load later, when Claude first reads a matching file. Editing one before it loads does take effect."）。

---

## 八、官方点名的"上下文相关失败模式"

`Best practices` 的 "Avoid common failure patterns" 一节，五个典型失败里三个都直接跟上下文有关：

| 失败模式 | 官方描述 | 官方解法 |
|---|---|---|
| **The kitchen sink session**（什么都往一个会话里塞） | "You start with one task, then ask Claude something unrelated, then go back to the first task. Context is full of irrelevant information." | "`/clear` between unrelated tasks."（任务之间 `/clear`） |
| **The over-specified CLAUDE.md**（CLAUDE.md 写太长） | "If your CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise."（太长会让 Claude 无视其中一半，重要规则淹没在噪音里） | "Ruthlessly prune."（无情精简；Claude 不做也对的内容就删掉或改成 hook） |
| **The infinite exploration**（无限探索） | "You ask Claude to 'investigate' something without scoping it. Claude reads hundreds of files, filling the context."（不限定范围就让它调研，读几百个文件灌满上下文） | "Scope investigations narrowly or use subagents so the exploration doesn't consume your main context."（收窄调研范围，或用 subagent 让探索不占主上下文） |

另一处反复强调的对应教训（`Best practices` "Course-correct early"）：

> "If you've corrected Claude more than twice on the same issue in one session, the context is cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt that incorporates what you learned. A clean session with a better prompt almost always outperforms a long session with accumulated corrections."
> （同一问题在同一会话里改了两轮还没好，说明上下文里塞满了失败路径。运行 `/clear`，带着学到的东西用更具体的 prompt 重来。**干净会话 + 更好的 prompt 几乎总胜过塞满修正的长会话。**）

本质都是同一个逻辑：**上下文里攒的噪音越多，模型越容易犯错，及时清空胜过硬撑着延续。**

---

## 九、落地清单：从官方口径到"我该怎么管上下文"

把八份文档的意思收拢成一张可执行表：

| 场景 | 官方建议 | 关键理由 |
|---|---|---|
| 随时看什么在占空间、拿优化建议 | **`/context`** | 按类别给出实时占用 + 优化建议，含已加载的 CLAUDE.md / auto memory |
| 切换到无关任务 | **`/clear`** | 旧对话挤掉下次需要的文件，还每条消息烧 token |
| 开始一个长新任务前 | **`/compact focus on ...`** | 保留你选的，而不是自动压缩猜的 |
| 同一问题改了两轮还没对 | **`/clear` + 更具体的 prompt 重来** | 干净会话 + 好 prompt 几乎总胜过塞满修正的长会话 |
| 想控制压缩保留什么 | CLAUDE.md 里加 **"Compact Instructions"** 小节 | 官方明确支持的持久化方式 |
| 只想压缩对话的一部分 | `Esc Esc` → `/rewind` → **Summarize from / up to here** | 保留早前完整 或 保留最近完整 |
| 快速小问题不想进上下文 | **`/btw`** | 答案进浮层，永不进对话历史 |
| 想让占用常驻可见 | 状态栏显示上下文用量 | 持续监控，而非事后才发现 |
| 大文件读取、跑测试、抓文档 | **subagent 隔离** | 6,100 token 读取只回 420 token 摘要 |
| CLAUDE.md 太大 | **精简到 200 行以内**，专项指令移进 skills / 路径规则 | 过长会被 Claude 无视一半；专项指令不必常驻 |
| 想省缓存失效的钱 | 模型/effort 开头定好；`/compact` 留在任务自然停顿处 | 中途改动会整段重读、零缓存命中 |
| 对话真的大到"管不过来" | 选 **1M token 扩展上下文**模型 | "需要更大的窗口而不是更小的对话"时 |

最后记住官方贯穿始终的两条红线：

1. **自动压缩只保"请求 + 关键代码片段"，保不住会话早期的详细指令**——所以持久规则必须写进 CLAUDE.md（根文件），而不是依赖聊天记录。压缩后，项目根 CLAUDE.md 和 auto memory 会从磁盘自动重生，路径规则和嵌套 CLAUDE.md 则会"丢到下次读到匹配文件"。
2. **上下文管理不是靠某一条命令，而是自动层（auto-compact / prompt caching / MCP 延迟加载）与手动层（`/context` `/clear` `/compact` `/rewind` `/btw` + subagent 隔离 + CLAUDE.md 精简）的组合**。它们的共同目标只有一个：让"窗口很快会满、满了会变笨"这条根本约束不咬到你。

---

## 参考来源

本文内容综合以下官方资料整理（**均于 2026-08-04 通过 web_fetch / curl 获取官方 `.md` 原文**；页面定位借助全站索引 https://code.claude.com/docs/llms.txt 关键词扫描）：

- **Glossary** — https://code.claude.com/docs/en/glossary
  （"Context window" 与 "Compaction" 的官方定义）
- **How Claude Code works** — https://code.claude.com/docs/en/how-claude-code-works
  （上下文窗口定义、"When context fills up"自动压缩机制、Compact Instructions、subagent 上下文隔离、"agentic harness / context management" 原话、thrashing 报错）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （交互式时间轴：启动加载项与示意 token、文件读取成本、subagent 的 6,100→420 token 节省量、"What survives compaction"存活表、手动 `/compact` `/clear`、1M 窗口、"Token counts are illustrative" 原话）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"The context window is the most important resource to manage"、"performance degrades as it fills" 等核心原话；Manage context aggressively 一节：`/clear`、`/compact <instructions>`、`/btw`、`/rewind` 部分压缩、subagent 调查；三个上下文失败模式）
- **Manage costs effectively** — https://code.claude.com/docs/en/costs
  （"Token costs scale with context size"；Reduce token usage 全策略：`/clear`、Compact Instructions、CLI 工具 vs MCP、hooks 预处理、skill 卸载 CLAUDE.md；"为什么长会话耗 token"机制；`/compact` 自身的成本）
- **How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （CLAUDE.md 会话启动即全量加载并耗 token、200 行建议、MEMORY.md 前 200 行/25KB 加载上限、压缩后项目根 CLAUDE.md 存活与嵌套 CLAUDE.md 丢失）
- **How Claude Code uses prompt caching** — https://code.claude.com/docs/en/prompt-caching
  （请求重发完整上下文机制、三层前缀缓存、使缓存失效的动作清单、"中途改 CLAUDE.md 不生效"、"Pick your model and effort level at the top of a session" 原话）
- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （"Understand context costs"：各特性上下文成本对照表）
- **Model configuration** — https://code.claude.com/docs/en/model-config
  （"Extended context" 一节：1M 支持的模型、`[1m]` 后缀用法、按套餐可用性、Sonnet 5 天生 1M 及其约 967K 自动压缩阈值、1M 不加价、LLM gateway / `CLAUDE_CODE_DISABLE_1M_CONTEXT` 退回 200K）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 隔离上下文的使用判据）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（大代码库上下文组织：分层 CLAUDE.md + LSP + subagent）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（提示词层面对抗上下文噪声）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
