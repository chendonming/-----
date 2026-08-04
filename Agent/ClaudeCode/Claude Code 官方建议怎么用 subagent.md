# Claude Code 官方建议怎么用 subagent？

> **一句话总结**：**subagent 的核心价值是"隔离上下文"——把会产生大量搜索/日志/文件内容、但你不需要长期留在主对话里的边角任务，丢进一个独立的上下文窗口去干，只把摘要拿回来。** 官方给出的使用判据很明确：**任务会输出你不想要的大段内容、你想施加工具/权限约束、或任务自成一体可以只返回摘要时，用 subagent**；反之，需要频繁来回迭代、多阶段共享上下文、快速小改动、或对延迟敏感的任务，用主对话。本文还整理了常被混淆的几组边界：subagent 与 skill 的区别、subagent 与 agent teams / dynamic workflows 的选型、subagent 嵌套与配额，以及定义 subagent 的官方最佳实践。
>
> 本文基于 Claude Code 官方文档 `Create custom subagents`、`Best practices for Claude Code`、`Explore the context window`、`Extend Claude Code`、`Run agents in parallel`、`Agent view`、`Orchestrate subagents at scale with dynamic workflows`、`Orchestrate teams of Claude Code sessions`，以及官方博客 `Steering Claude Code` 整理，文末附参考来源。

subagent（子代理）是 Claude Code 里最常见也最容易被用偏的一个功能。很多人要么完全不用、什么都让主对话干，要么一上来就堆一堆 subagent、把上下文塞得更满。

官方文档对"怎么用 subagent"其实给了非常具体的判据。下面按官方口径逐条拆开。

---

## 一、先搞懂：官方把 subagent 定位成什么

`Create custom subagents` 文档开头一句话就定义了它：

> "Subagents are specialized AI assistants that handle specific types of tasks."
> （subagent 是处理特定类型任务的专业化 AI 助手。）

紧接着给了一句**最重要的使用判据**，几乎全文都在围绕它展开：

> "Use one when a side task would flood your main conversation with search results, logs, or file contents you won't reference again: the subagent does that work in its own context and returns only the summary."
> （当一个**边角任务**会把搜索结果、日志、或你再也不会引用的文件内容灌进你的主对话时，就派一个 subagent 去干：它**在自己的上下文里**完成工作，**只把摘要返回**。）

以及第二个判据——针对"重复造轮子"的情况：

> "Define a custom subagent when you keep spawning the same kind of worker with the same instructions."
> （当你反复用相同的指令生成同一种工人时，就把它定义成自定义 subagent。）

注意这里的关键词：**side task（边角任务）**。官方明确说 subagent 是为了让"探索/实现的噪音"离开主对话。它的架构是：

> "Each subagent runs in its own context window with a custom system prompt, specific tool access, and independent permissions."
> （每个 subagent 运行在**独立上下文窗口**里，带自定义系统提示词、特定的工具访问权限和独立的权限配置。）

官方博客 `Steering Claude Code` 把这层隔离讲得更直白——subagent 正文里的指令不仅不会自动加载，甚至**根本不会进入父会话**：

> "Not only does the larger instructional context within the body of the subagent not auto-invoke, it never enters the parent conversation at all."
> （subagent 正文里更大的指令上下文不仅不会自动被调用，而且**根本不会进入父会话**。）

> "The subagent then runs in its own fresh context window."
> （subagent 随后在它自己全新的上下文窗口里运行。）

官方文档把 subagent 的价值概括为五条，其中第一条就是上下文：

| 官方原话 | 中文 | 一句话解释 |
|---|---|---|
| **Preserve context** by keeping exploration and implementation out of your main conversation | 保留上下文 | 探索和实现都发生在子上下文里，主对话不涨 |
| **Enforce constraints** by limiting which tools a subagent can use | 强制约束 | 用 `tools` 限制它能碰什么，天然形成护栏 |
| **Reuse configurations** across projects with user-level subagents | 复用配置 | 放在 `~/.claude/agents/` 即可全局复用 |
| **Specialize behavior** with focused system prompts for specific domains | 专业化行为 | 每个 subagent 一套专属系统提示词 |
| **Control costs** by routing tasks to faster, cheaper models like Haiku | 控制成本 | 把任务路由到更便宜的小模型（如 Haiku） |

## 二、上下文节省量：官方给了一个很直观的数字

subagent 隔离上下文到底省多少？`Explore the context window` 文档里有一段交互式模拟，专门演示一个研究型 subagent 的上下文走向，文末的结论是一个很震撼的对比：

> "The subagent read 6,100 tokens of files. You got a 420-token result. That's the context savings."
> （subagent 读了 **6,100 token** 的文件，你只拿到 **420 token** 的结果。**这就是上下文节省量。**）

这 6,100 token 的文件内容全部留在 subagent 自己的上下文里，主对话只收进一条 420 token 的最终回复——大约 **1/15**。官方在 `Best practices for Claude Code` 文档里点明："The context window is the most important resource to manage"（上下文窗口是最需要管理的资源），subagent 就是官方给出的**管理手段之一**，而且被列为"最有力的工具"之一（见下节）。

## 三、官方在 best-practices 里的定位：subagent 是"最有力的工具"

`Best practices for Claude Code` 文档在 "Use subagents for investigation" 一节里，给了比功能文档更直白的定位：

> "Since context is your fundamental constraint, subagents are one of the most powerful tools available."
> （既然上下文是你的根本约束，subagent 就是可用工具里**最有力**的一个。）

同节给出的标准用法：让 subagent 去**调研**，把文件阅读量挡在主对话之外——官方示例：

> "Use subagents to investigate how our authentication system handles token refresh, and whether we have any existing OAuth utilities I should reuse."
> （用 subagent 去调研我们的认证系统如何处理 token 刷新，以及有没有可复用的 OAuth 工具。）

以及**验证**用途——实现完之后用独立上下文复查：

> "use a subagent to review this code for edge cases"
> （用 subagent 审查这段代码的边界情况。）

best-practices 的"常见失败模式"一节里，甚至把"无限探索"列成一个典型坑，解法之一就是 subagent：

> "The infinite exploration. You ask Claude to 'investigate' something without scoping it. Claude reads hundreds of files, filling the context. Fix: Scope investigations narrowly or use subagents so the exploration doesn't consume your main context."
> （**无限探索**：你不限定范围就让它"调研"，Claude 读了几百个文件，灌满上下文。**解法**：把调研范围收窄，或用 subagent，让探索不消耗主上下文。）

还有一条容易被忽视的**对抗式复查**建议：任务结束前，让一个**全新上下文**的 subagent 复查 diff，而不是让"干活的人给自己打分"：

> "Before treating a task as done, have a subagent review the diff in a fresh context and report gaps."
> （在把任务当成完成之前，让一个全新上下文里的 subagent 复查 diff 并报告差距。）

## 四、官方判据：什么时候用 subagent，什么时候留在主对话

这是官方文档专门列出的对照（`Create custom subagents` → "Choose between subagents and main conversation"），是"怎么用"最核心的一张表：

| 场景 | 官方建议 | 官方理由 |
|---|---|---|
| 任务需要频繁来回、迭代打磨 | **主对话** | "The task needs frequent back-and-forth or iterative refinement" |
| 多个阶段共享大量上下文（规划→实现→测试） | **主对话** | "Multiple phases share significant context" |
| 快速、定点的小改动 | **主对话** | "You're making a quick, targeted change" |
| 对延迟敏感 | **主对话** | "Latency matters. Subagents start fresh and may need time to gather context"（subagent 冷启动、需要时间收集上下文） |
| 任务会产生你主上下文不需要的大段输出 | **subagent** | "The task produces verbose output you don't need in your main context" |
| 想施加特定的工具/权限限制 | **subagent** | "You want to enforce specific tool restrictions or permissions" |
| 任务自成一体、能返回一个摘要 | **subagent** | "The work is self-contained and can return a summary" |

注意"对延迟敏感用主对话"这条——很多人以为 subagent 总更快，其实官方明说它会**从头冷启动**，还要花时间收集上下文，所以对延迟敏感的小事反而别用。

## 五、官方给出的三个典型用法

`Create custom subagents` 文档 "Common patterns" 一节，给了三种"正确姿势"，这是最能直接照抄的部分：

### 1. 隔离高流量操作（isolate high-volume operations）

官方称之为"最有效的用途之一"：

> "One of the most effective uses for subagents is isolating operations that produce large amounts of output. Running tests, fetching documentation, or processing log files can consume significant context. By delegating these to a subagent, the verbose output stays in the subagent's context while only the relevant summary returns to your main conversation."
> （subagent **最有效的用途之一**，是隔离会产生大量输出的操作。跑测试、抓文档、处理日志都会吃掉大量上下文。把这些派给 subagent 后，啰嗦的输出留在它的上下文里，只有相关摘要回到主对话。）

官方示例 prompt：

> "Use a subagent to run the test suite and report only the failing tests with their error messages"
> （用 subagent 跑测试套件，只汇报失败的测试和错误信息。）

### 2. 并行研究（run parallel research）

对彼此独立的调研，拆成多个 subagent 同时跑：

> "For independent investigations, spawn multiple subagents to work simultaneously"
> （对相互独立的调研，同时生成多个 subagent 一起干。）

配套的注意点是，它要求"研究路径互不依赖"：

> "Each subagent explores its area independently, then Claude synthesizes the findings. This works best when the research paths don't depend on each other."
> （每个 subagent 独立探索自己的区域，然后 Claude 综合结论。这在**各研究路径互不依赖**时效果最好。）

### 3. 链式 subagent（chain subagents）

多步流程里让 subagent 一个接一个，上一步的结论传给下一步：

> "For multi-step workflows, ask Claude to use subagents in sequence. Each subagent completes its task and returns results to Claude, which then passes relevant context to the next subagent."
> （对多步工作流，让 Claude **按顺序**使用 subagent。每个 subagent 完成任务后把结果交给 Claude，Claude 再把相关上下文传给下一个。）

这三种模式背后有一个共同的官方告诫，别把"并行"用成"灌水"：

> "When subagents complete, their results return to your main conversation. Running many subagents that each return detailed results can consume significant context."
> （subagent 完成后，结果会回到你的主对话。**跑很多各返回详细结果的 subagent 也会消耗大量上下文。**）

也就是说：subagent 隔离的是"过程"，不是"结果"；如果每个 subagent 都吐出大段结果，主对话照样会被灌满。

## 六、边界在哪：subagent vs agent teams vs dynamic workflows

"怎么用 subagent"绕不开一个问题——**什么时候用 subagent，什么时候该升级到别的并行机制**。官方 `Run agents in parallel` 文档把四种方案放在一张表里对照：

| 方案 | 官方定位 | 什么时候用 |
|---|---|---|
| **Subagents** | 单一会话内的委派工人，在自己的上下文里干边角活、返回摘要 | "A side task would flood your main conversation with search results, logs, or file contents you won't reference again" |
| **Agent view** | 一个屏幕调度/监控多个后台会话（`claude agents`） | 你有几个独立任务要交出去，随后回来看状态 |
| **Agent teams** | 多个协调的会话，共享任务列表、agent 间直接通信，由 lead 管理（实验性、默认关闭） | 想让 Claude 把项目拆成几块、分配下去、并让工人保持同步 |
| **Dynamic workflows** | 一个脚本编排大量 subagent 并交叉验证结果 | 任务超出"一把 subagent"能协调的规模，或需要结果互相验证 |

关键判据在 `workflows.md` 的 "When to use a workflow" 一节，官方用"**谁握着计划**"来区分：

> "With subagents, skills, and agent teams, Claude is the orchestrator: it decides turn by turn what to spawn or assign next, and every result lands in a context window. A workflow script holds the loop, the branching, and the intermediate results itself, so Claude's context holds only the final answer."
> （用 subagent、skills、agent teams 时，**Claude 是编排者**：它逐轮决定接下来生成/分配什么，每个结果都落进上下文窗口。而 workflow 脚本自己持有循环、分支和中间结果，Claude 的上下文只装最终答案。）

规模维度上官方给的数字也很明确：

| | Subagents | Workflows |
|---|---|---|
| 规模 | "A few delegated tasks per turn"（每轮几个委派任务） | "Dozens to hundreds of agents per run"（每次运行几十到几百个 agent） |

至于 agent teams，官方明确说它和 subagent 的取舍标准是"工人之间**要不要互相通信**"：

> "Use subagents when you need quick, focused workers that report back. Use agent teams when teammates need to share findings, challenge each other, and coordinate on their own."
> （当你需要**快速、专注、只回报结果**的工人时用 subagent；当队友需要**共享发现、互相质疑、自行协调**时用 agent teams。）

而且官方**明确提醒 agent teams 有协调开销**，subagent 反而是更轻的选择：

> "Agent teams add coordination overhead and use significantly more tokens than a single session. They work best when teammates can operate independently. For sequential tasks, same-file edits, or work with many dependencies, a single session or subagents are more effective."
> （agent teams 有协调开销，且 token 消耗显著高于单会话。它们只在队友能独立工作时最好。对**顺序任务、同文件编辑、或依赖很多的活，单会话或 subagent 更有效**。）

一句话总结官方的梯子：**单会话 → subagent → agent teams / workflows**。subagent 是"一把子活的轻量委派"，再往上才需要考虑更重的编排机制。

另外还有一个官方已经"打包好"的 subagent 玩法值得知道——`/batch`。`Run agents in parallel` 文档把它定位成：

> "`/batch` is a skill that has Claude split one large change into 5 to 30 worktree-isolated subagents that each open a pull request. It's a packaged use of subagents and worktrees, not a separate coordination style."
> （`/batch` 是一个 skill：让 Claude 把一个大的改动拆成 **5–30 个 worktree 隔离的 subagent**，每个各开一个 PR。它是 **subagent 和 worktree 的打包用法**，不是另一种协调风格。）

也就是说，如果你要的是"一次大改动、拆成并行的隔离任务、各自出 PR"，`/batch` 就是官方替你排好的 subagent 方案，不用自己编排。

和 subagent 并列还有一个容易被绕进去的选项——**skill**。官方在 "Choose between subagents and main conversation" 一节末尾专门给了一条界线：

> "Consider Skills instead when you want reusable prompts or workflows that run in the main conversation context rather than isolated subagent context."
> （当你想要可复用的提示词或工作流、且希望它们在**主对话上下文**里运行，而不是隔离的 subagent 上下文时，改用 Skills。）

一句话区分：**subagent 是"隔离到子上下文去干"，skill 是"把可复用指令塞进主对话去干"**——同是可复用单元，差别在"隔离不隔离"。

`Extend Claude Code` 文档给了一张更系统的对照表：

| 维度 | Skill | Subagent |
|---|---|---|
| **本质** | Reusable instructions, knowledge, or workflows（可复用的指令/知识/流程） | Isolated worker with its own context（有独立上下文的隔离工人） |
| **核心收益** | Share content across contexts（跨上下文共享内容） | Context isolation（上下文隔离）——工作独立发生，只有摘要返回 |
| **对上下文窗口的影响** | Adds to your main window（**加进主窗口**） | Uses a separate window with its own input and output tokens（**用独立窗口**） |
| **最适合** | Reference material, invocable workflows（参考资料、可调用流程） | Tasks that read many files, parallel work, specialized workers（读大量文件的任务、并行工作、专用工人） |

官方博客给了一句比表格更好记的判据：

> "Use a skill when you want the procedure to play out inside the main thread so you can see and steer each step."
> （当你想让流程**在主线程里逐步展开**、以便亲眼看到并干预每一步时，用 skill。）

也就是说：**流程你想盯着看 → skill；流程结果你只关心摘要 → subagent。** `Extend Claude Code` 的"渐进搭建"（build your setup over time）表格里，也把 subagent 的触发时机归纳为一句话：当一个旁路任务"把你不打算再引用的输出灌进对话"时（"A side task floods your conversation with output you won't reference again"），就该把它路由给 subagent。

## 七、定义一个好 subagent：官方四条最佳实践

`Create custom subagents` 文档在示例一节开头给了四条"设计铁律"（Best practices tip）：

> - **Design focused subagents:** each subagent should excel at one specific task
> - **Write detailed descriptions:** Claude uses the description to decide when to delegate
> - **Limit tool access:** grant only necessary permissions for security and focus
> - **Check into version control:** share project subagents with your team

翻译并展开：

| 官方建议 | 含义 | 落地做法 |
|---|---|---|
| **Design focused** | 每个 subagent 只精一件事 | 一个"代码审查者"别同时干"调数据库"；职责越单一越容易被正确委派 |
| **Write detailed descriptions** | description 是 Claude 决定"何时委派"的依据 | "Claude uses each subagent's description to decide when to delegate tasks"——description 要写清**什么场景该用**，甚至可以写 "Use proactively after writing or modifying code" 这类触发词 |
| **Limit tool access** | 只给必要权限，安全 + 专注 | 只读审查员就给 `Read, Grep, Glob`，把 `Write/Edit` 排除；`tools` 字段本身就是护栏 |
| **Check into version control** | 项目级 subagent 提交进仓库 | 放 `.claude/agents/` 并 git 提交，团队共享、共同改进 |

文件形式是一份带 YAML frontmatter 的 Markdown。官方文档的 walkthrough 让你用一句自然语言创建，最终落在 `~/.claude/agents/code-improver.md` 的文件长这样：

```markdown ~/.claude/agents/code-improver.md
---
name: code-improver
description: Scans files and suggests improvements for readability, performance, and best practices. Use after writing or modifying code.
tools: Read, Grep, Glob
model: sonnet
---

You are a code improvement specialist. For each issue you find, explain
the problem, show the current code, and provide an improved version.
```

注意这个例子里的 `description` 末尾写着 **"Use after writing or modifying code"**——这就是"写详细 description"的实战形态：不只在介绍这个 subagent 是什么，还告诉 Claude **什么场景该触发它**。

定义时还能给 subagent **预装 skills**。官方在 `Extend Claude Code` 文档里说明：

> "Skills listed in the subagent's `skills` field are fully preloaded into its context at launch."
> （写在 subagent `skills` 字段里的 skills，会在启动时**完整预加载**进它的上下文——不像主对话那样按需加载。）

反向也成立，skill 同样可以借用 subagent 的隔离上下文：官方原话 "A skill can run in isolated context using `context: fork`"（一个 skill 也可以用 `context: fork` 在隔离上下文里运行）。一句话：**subagent 与 skill 可以互相配合**——subagent 预装 skill 获得专长，skill 借 subagent 获得隔离。

官方文档还专门提示：subagent 文件可以放在不同作用域——项目级 `.claude/agents/`、用户级 `~/.claude/agents/`、插件级、CLI 级，优先级从高到低依次为 managed settings > `--agents` > 项目 > 用户 > 插件。

## 八、三种调用方式 & 内置 subagent

`Create custom subagents` 文档 "Invoke subagents explicitly" 一节说，除了让 Claude **自动委派**（根据你的请求、subagent 的 description、当前上下文决定），你还可以自己点名，从"一次性"到"整会话"分三档：

| 方式 | 力度 | 说明 |
|---|---|---|
| **自然语言** | 一次建议 | "Use the test-runner subagent to fix failing tests"，Claude 决定是否委派 |
| **@-mention** | 保证一次执行 | `@"code-reviewer (agent)" look at the auth changes`，确保跑这个 subagent 而不是让 Claude 选 |
| **`--agent` / `agent` 设置** | 整会话生效 | `claude --agent code-reviewer`，整个会话都用它的系统提示词、工具限制和模型 |

内置的 subagent 有三个主力（Claude Code 会自动按场景使用）：

| 内置 subagent | 官方定位 | 关键属性 |
|---|---|---|
| **Explore** | "A fast, read-only agent optimized for searching and analyzing codebases" | 只读（无 Write/Edit）、专门做文件发现/代码搜索/代码库探索；与 Plan 一同跳过 CLAUDE.md 与 git status，以保持快速低成本 |
| **Plan** | "A research agent used during plan mode to gather context before presenting a plan" | 只读，plan 模式下收集上下文的调研员（与 Explore 一同跳过 CLAUDE.md 与 git status） |
| **general-purpose** | "A capable agent for complex, multi-step tasks that require both exploration and action" | 工具全开，需要"既探索又动手"的复杂多步任务 |

还有一个**背景/前台**的细节值得知道：自 v2.1.198 起，subagent **默认在后台运行**——"As of v2.1.198, subagents run in the background by default. Claude runs a subagent in the foreground when it needs the result before continuing."（当 Claude 需要结果才能继续时才以前台运行。）后台 subagent 能让你继续干别的活，但也请注意它有更小的内置工具集。

### subagent 还能派 subagent：嵌套、配额，以及一个常被误用的替代

官方文档专门回答了"subagent 能不能套娃"——**能**，默认允许嵌套三层：

> "By default, a subagent can spawn subagents of its own, up to three layers below the main conversation."
> （默认情况下，subagent 可以再派出自己的 subagent，最多到主对话以下的**三层**。）

嵌套的意义是**逐层隔离中间输出**——官方举的例子：一个审查者 subagent 为每条发现再派一个验证者 subagent，中间产物全程不进入主对话，只有最顶层的摘要回到你这里。注意，**嵌套会吃掉主对话的委派预算**（见下表），且同一时刻有数量上限。

配合并行与嵌套，官方给了两个**硬性配额**，防止 subagent 失控：

| 配额 | 默认值 | 官方原话 | 配置项 |
|---|---|---|---|
| **会话内 spawn 总数** | 200 个 | "By default, Claude can spawn at most 200 subagents per session." | `CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION`（v2.1.212+） |
| **同时运行上限** | 20 个 | "By default, when 20 subagents are running in a session, spawning another with the Agent tool fails with 'Concurrent subagent limit reached'." | `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`（v2.1.217+） |

另外，对"只是问一句会话里已有的内容"这种场景，官方给的**不是** subagent，而是 `/btw`——这个边界很多人会搞混：

> "For a quick question about something already in your conversation, use /btw instead of a subagent. It sees your full context but has no tool access, and the answer is discarded rather than added to history."
> （对会话里已有内容的快速提问，用 `/btw` 而不是 subagent——它能看到你的完整上下文，但**没有工具权限**，答案也不会写进历史。）

## 九、落地建议：从官方口径到"我该怎么用"

把官方六篇文档的意思收拢成一张可执行清单：

| 场景 | 官方建议 | 关键理由 |
|---|---|---|
| 跑测试、抓文档、处理日志这类"大输出"活 | **用 subagent**，只拿摘要 | 官方称之为"最有效的用途之一" |
| 独立调研（认证/数据库/API 分头查） | **并行 subagent** | 路径互不依赖时才有效 |
| 快速小改动、需要频繁迭代 | **留在主对话** | subagent 冷启动 + 收集上下文有延迟 |
| 代码实现完复查边界情况 / 复查 diff | **subagent 独立复查** | 全新上下文，避免"自己给自己打分" |
| 需要工人之间互相讨论、质疑、协调 | **才考虑 agent teams** | 协调开销大、token 显著更高，官方默认关闭 |
| 规模大到"一把 subagent"协调不过来、或要结果交叉验证 | **才考虑 dynamic workflows** | "Dozens to hundreds of agents per run" |
| 反复用相同指令生成同一种工人 | **定义自定义 subagent** | 复用 + 专精 + 工具护栏 |

最后记住官方反复强调的那条红线：**subagent 省的是"过程"，不是"结果"**。多个各返回大段结果的 subagent，照样会把主上下文灌满——官方原文："Running many subagents that each return detailed results can consume significant context."

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch / 抓取官方 Markdown 原文 `code.claude.com/docs/en/*.md` 核对获取）：

- **Create custom subagents** — https://code.claude.com/docs/en/sub-agents
  （本文主干：subagent 定义、使用判据、内置 subagent、调用方式、三种典型模式、主对话 vs subagent 对照、设计最佳实践；另含 subagent vs skill 界线、嵌套与配额——200/会话、20 并发、`/btw` 边界）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"subagents are one of the most powerful tools"、调研/验证/对抗式复查用法、"无限探索"失败模式与 subagent 解法、自定义 subagent 示例）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （6,100 token 文件读取 → 420 token 摘要的上下文节省量演示，"That's the context savings"）
- **Run agents in parallel** — https://code.claude.com/docs/en/agents
  （subagents / agent view / agent teams / dynamic workflows 四方案对照表与选型判据；`/batch` 打包用法：5–30 个 worktree 隔离 subagent）
- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （subagent 定位"isolated execution context that returns summarized results"、Skill vs Subagent 对照、subagent 的 `skills:` 字段预加载、`context: fork` 隔离运行、渐进搭建触发时机）
- **Agent view / background agents** — https://code.claude.com/docs/en/agent-view
  （后台会话与 subagent 的区别：独立会话 vs 会话内委派；`claude agents` 调度/监控界面）
- **Claude 官方博客：Steering Claude Code — when to use CLAUDE.md, skills, hooks, and subagents** — https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more
  （subagent 定义与"正文不进入父会话"机制；"Use a skill when you want the procedure to play out inside the main thread" 判据；deep search / log analysis / dependency audit 示例）
- **Orchestrate subagents at scale with dynamic workflows** — https://code.claude.com/docs/en/workflows
  （subagents vs skills vs agent teams vs workflows 对照、"谁握着计划"、规模数字：每轮几个 vs 每轮几十到几百）
- **Orchestrate teams of Claude Code sessions** — https://code.claude.com/docs/en/agent-teams
  （subagent vs agent teams 取舍："quick, focused workers" vs "share findings, challenge each other"、协调开销与 token 成本）

> 相关文档：`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（subagent 在大代码库中承担探索隔离角色的定位）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口管理）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
