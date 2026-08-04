# Claude Code 官方文档对上下文窗口和上下文管理怎么说

> **一句话总结**：官方把上下文窗口定义为**装下你整个会话一切信息的地方**——对话历史、读过的文件、命令输出、CLAUDE.md、自动记忆、已加载的 skill 和系统指令；它**填得很快、满了性能会下降**，因此官方原话是"**The context window is the most important resource to manage（上下文窗口是最需要管理的资源）**"。管理手段分两层：**自动层**（接近上限时先清旧工具输出、再自动压缩对话；压缩后项目根 CLAUDE.md 和自动记忆会从磁盘重新注入，但早期对话中的细节会丢）+ **手动层**（`/context` 查看占用、`/compact` 带焦点压缩、`/clear` 换任务清零、subagent 把大文件读取隔离到独立窗口、skill/MCP 按需加载）。想要"更大窗口"而不是"更小对话"，官方给出的答案是 Fable 5 / Sonnet 5 / Opus 4.6 及之后 / Sonnet 4.6 的 **100 万 token 上下文**。
>
> 本文基于 Claude Code 官方文档 `How Claude Code works`、`Explore the context window`、`Best practices for Claude Code`、`Manage costs effectively`、`How Claude remembers your project`、`How Claude Code uses prompt caching` 六份资料整理，文末附参考来源与获取方式。

---

## 一、官方怎么定义"上下文窗口"

两份文档给出了几乎一致的官方定义。`How Claude Code works` 的原文：

> "Claude's context window holds your conversation history, file contents, command outputs, CLAUDE.md, auto memory, loaded skills, and system instructions."
> （Claude 的上下文窗口装着你的对话历史、文件内容、命令输出、CLAUDE.md、自动记忆、已加载的 skill 和系统指令。）

`Best practices` 的说法更强调"整个过程都在里面"：

> "Claude's context window holds your entire conversation, including every message, every file Claude reads, and every command output."
> （Claude 的上下文窗口装着你的整个对话，包括每一条消息、Claude 读过的每一个文件、每一条命令输出。）

关键点：**上下文窗口是"整段会话"级别的**——它不是一个短暂的缓存，而是 Claude Code 每次调用模型时都会跟着发送的完整历史（`prompt-caching` 文档的底层解释）：

> "Claude Code re-sends the full context: the system prompt, your project context, every prior message and tool result, and your new message."
> （Claude Code 会重新发送完整的上下文：系统提示词、项目上下文、此前的每一条消息和工具结果，以及你的新消息。）

### 窗口有多大

官方没有给一个"统一大小"的硬数字，而是分两档：

| 窗口 | 说明 |
|---|---|
| **默认窗口（约 20 万 token）** | `Explore the context window` 这个交互式模拟页以 `MAX = 200000`（约 200K token）演示会话如何被填满，但页面也注明 **"Token counts are illustrative"（token 数是示意性的）**，实际值随你的 CLAUDE.md 大小、MCP server 数量和文件长度而变。 |
| **扩展窗口（100 万 token）** | `context-window` 文档原话：**"Fable 5, Sonnet 5, Opus 4.6 and later, and Sonnet 4.6 support a 1 million token context window."**（Fable 5、Sonnet 5、Opus 4.6 及之后、Sonnet 4.6 支持 100 万 token 的上下文窗口。）"Compaction works the same way at the larger limit."（压缩在更大的上限下工作方式相同。） |

官方在"想要更大窗口"上的态度很明确——**先考虑把对话变小，而不是直接换大窗口**（见第五节）。

---

## 二、为什么上下文是"最需要管理的资源"

`Best practices` 把整个最佳实践体系的出发点，归结为这一条约束：

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."
> （大多数最佳实践都基于同一条约束：Claude 的上下文窗口**填得很快**，而且**随着它被填满，性能会下降**。）

官方给了两个直接原因：

1. **填充速度超出直觉**：
   > "A single debugging session or codebase exploration might generate and consume tens of thousands of tokens."
   > （一次调试会话或代码库探索，就可能产生并消耗数万 token。）

2. **填满后是真的会"变笨"**：
   > "When the context window is getting full, Claude may start 'forgetting' earlier instructions or making more mistakes."
   > （当上下文窗口快满时，Claude 可能开始"忘记"更早的指令，或犯更多错误。）

由此引出整份文档里最关键的一句原话：

> "**The context window is the most important resource to manage.**"
> （**上下文窗口是最需要管理的资源。**）

`How Claude Code works` 甚至把"上下文管理"直接写进了 Claude Code 的定位——它不是一个普通聊天外壳，而是：

> "Claude Code serves as the **agentic harness** around Claude: it provides the tools, context management, and execution environment…"
> （Claude Code 是包在 Claude 外面的**智能体 harness**：它提供工具、**上下文管理**和执行环境……）

---

## 三、上下文里到底装了什么

`Explore the context window` 页面用一个交互式时间轴演示了一场会话：从你输入第一个字之前，到 `/compact` 之后。它把"上下文里有什么、各占多少"拆成了看得见的清单（token 数均为示意值，但**相对量级**很能说明问题）：

### 3.1 在你输入任何内容之前（启动即加载）

| 加载项 | 示意 token | 官方说明要点 |
|---|---|---|
| **System prompt（系统提示词）** | ~4,200 | "Always loaded first. You never see it."（永远最先加载，你永远看不到它。） |
| **Auto memory（MEMORY.md 自动记忆）** | ~680 | 只加载 **前 200 行或前 25KB**（先到者为准）。 |
| **Environment info（环境信息）** | ~280 | 工作目录、平台、shell、OS、是否 git 仓库；git 分支/状态/近期提交作为独立块放在系统提示词末尾。 |
| **MCP tools（默认延迟加载）** | ~120 | 默认只列出工具名，完整 schema 按需经 tool search 加载。 |
| **Skill descriptions（skill 描述）** | ~450 | 只加载一行描述；完整内容等到真正调用某个 skill 才加载。且**压缩后不会重新注入**。 |
| **~/.claude/CLAUDE.md（用户级）** | ~320 | 你的全局偏好，每个项目、每次对话启动都加载。 |
| **Project CLAUDE.md（项目级）** | ~1,800 | 官方最强调的文件；提示语："Keep it under 200 lines."（控制在 200 行以内。） |

官方在这个时间轴里给的最重要提示之一，就是**项目 CLAUDE.md 应该克制**：

> "Keep it under 200 lines. Move reference content to skills or path-scoped rules so it only loads when needed."
> （把它控制在 200 行以内。把参考资料挪到 skill 或路径作用域规则里，让它只在需要时加载。）

### 3.2 会话进行中，谁在吃掉上下文

时间轴显示，随着 Claude 工作，主要增量来自：

- **每次文件读取**（File reads dominate context usage——"文件读取主导上下文占用"）；
- 读取文件时**自动加载的路径作用域规则**（`.claude/rules/` 里带 `paths:` 的规则）；
- 每次编辑后触发的 **PostToolUse hook**（只有写入 `additionalContext` 的输出才进上下文）；
- 你的每一条后续提问（都会在同一份上下文上继续累加）。

### 3.3 会话之间是隔离的

> "Each new session starts with a fresh context window, without the conversation history from previous sessions."
> （每个新会话都以全新的上下文窗口开始，不含此前会话的对话历史。原话出自 `How Claude Code works` 的 "Work with sessions" 一节。）

跨会话的知识靠两个机制延续：你写的 **CLAUDE.md** 和 Claude 自己写的 **auto memory**（`memory` 文档原话）。两者都在每次对话启动时加载；但 auto memory 同样受"前 200 行 / 25KB"上限约束。

---

## 四、上下文满了怎么办：自动压缩 + 手动三板斧 + 更大窗口

### 4.1 自动压缩（auto-compaction）

`How Claude Code works` 描述了两级自动处理：

> "Claude Code manages context automatically as you approach the limit. It clears older tool outputs first, then summarizes the conversation if needed."
> （Claude Code 在你接近上限时自动管理上下文：**先清掉较早的工具输出，必要时再把对话总结成摘要**。）

注意它的取舍：

> "Your requests and key code snippets are preserved; detailed instructions from early in the conversation may be lost."
> （你的请求和关键代码片段会被保留；**早期对话中的详细指令可能会丢**。）

所以官方紧接着给了最重要的建议：**把要长期生效的规则写进 CLAUDE.md，而不是放在对话里**。

> "Put persistent rules in CLAUDE.md rather than relying on conversation history."
> （把持久规则放进 CLAUDE.md，而不是依赖对话历史。）

**抖动保护**：如果单个文件或工具输出太大，导致每次压缩完立刻又被填满，官方明确说 Claude Code 不会死循环：

> "Claude Code stops auto-compacting after a few attempts and shows an error instead of looping."
> （尝试几次后 Claude Code 会停止自动压缩并报错，而不是循环。）

### 4.2 手动三板斧：`/context`、`/compact`、`/clear`

| 命令 | 官方定位 | 关键原话 |
|---|---|---|
| **`/context`** | 查看上下文占用，获得优化建议 | "Run `/context` to see what's using space."（运行 `/context` 查看是什么在占空间。）"run `/context` for a live breakdown by category with optimization suggestions"（实时按类别分解，带优化建议。） |
| **`/compact`** | 手动把对话压缩成结构化摘要，**可带焦点** | "run `/compact` with a focus (like `/compact focus on the API changes`)"（带焦点运行 `/compact`，如 `/compact focus on the API changes`。）摘要保留"你的请求和意图、关键技术概念、看过/改过的文件及重要代码片段、错误及修复方式、待办任务、当前工作"。 |
| **`/clear`** | 换任务时清空上下文，从零开始 | "run `/clear` when switching to unrelated work. Old conversation crowds out the files you need next and costs tokens on every message."（切换到无关任务时运行 `/clear`。旧对话挤掉你接下来需要的文件，并在每条消息上耗 token。） |

`/compact` 与 `/clear` 的关键区别（`costs` 文档原话）：

> "When you want a fresh start instead of continuity, `/clear` costs nothing."
> （当你想要全新开始而不是延续时，`/clear` 不花 token。）

因为 `/compact` 本身是一次"读整个要压缩的对话"的大请求，而 `/clear` 只是清空。

### 4.3 压缩的"焦点"与自定义指令

有两种方式控制压缩时保留什么：

1. **命令行带焦点**：`/compact focus on the auth bug fix`——"The summary keeps what you choose instead of what the automatic pass guesses is important."（摘要保留你选择的内容，而不是自动压缩猜的内容。）
2. **在 CLAUDE.md 写 `# Compact instructions`**：
   > "To control what's preserved during compaction, add a 'Compact Instructions' section to CLAUDE.md or run `/compact` with a focus."
   > （要控制压缩时保留什么，在 CLAUDE.md 里加"Compact Instructions"一节，或带焦点运行 `/compact`。）

```markdown
# Compact instructions

When you are using compact, please focus on test output and code changes
```

### 4.4 想要更大窗口？

`context-window` 文档原话：

> "If you need a larger window rather than a smaller conversation, Fable 5, Sonnet 5, Opus 4.6 and later, and Sonnet 4.6 support a 1 million token context window."
> （如果你需要**更大的窗口**而不是**更小的对话**，Fable 5、Sonnet 5、Opus 4.6 及之后、Sonnet 4.6 支持 100 万 token 的上下文窗口。）

注意官方把"更大窗口"和"更小对话"并列为两条路，且**默认先谈怎么把对话变小**——窗口能换，但上下文管理的功夫该花还在花。

---

## 五、压缩之后：什么活着、什么会丢

`Explore the context window` 给了一张官方的"压缩存活表"（What survives compaction），是全文信息密度最高的一张表：

| 机制 | 压缩之后 |
|---|---|
| **System prompt 与 output style** | 不变（不属于消息历史） |
| **项目根 CLAUDE.md 与无作用域规则** | **从磁盘重新注入** |
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

## 六、官方给的上下文管理手段（按"省多少"排序）

把六份文档里所有"主动管理上下文"的手段归拢，是一张这样的表：

| 手段 | 官方原话/要点 | 省的是什么 |
|---|---|---|
| **subagent 隔离探索** | "[Subagents] get their own fresh context, completely separate from your main conversation. Their work doesn't bloat your context."（subagent 有自己全新的、与主对话完全独立的上下文，它们的工作不会撑大你的上下文。） | 大文件读取留在 subagent 的窗口里，只回传摘要。官方数字：**subagent 读了 6,100 token，你只收到 420 token 的结果——"That's the context savings."** |
| **skill 按需加载** | "Claude sees skill descriptions at session start, but the full content only loads when a skill is used."（启动时只看到描述，完整内容用到才加载。）`disable-model-invocation: true` 可让描述也不进上下文，直到你手动 `/name` 调用。 | 把不常用的领域知识从"常驻"变"按需"。 |
| **MCP 工具延迟加载（tool search）** | "MCP tool definitions are deferred by default… so only tool names consume context until Claude uses a specific tool."（默认延迟，工具名先占极小空间，用到才加载 schema。） | 每个 MCP server 的工具定义不再常驻。 |
| **路径作用域规则** | 带 `paths:` 的 `.claude/rules/` 规则，只在 Claude 读到匹配文件时加载。 | 指令不常驻，随用随加载。 |
| **提示词写具体** | "Be specific in prompts… so Claude reads fewer files."（提示词写具体，Claude 少读文件。）`context-window` 明说"File reads dominate context usage"（文件读取主导上下文占用）。 | 最贵的单项——文件读取。 |
| **hooks 预处理数据** | "Instead of Claude reading a 10,000-line log file to find errors, a hook can grep for `ERROR` and return only matching lines."（与其让 Claude 读 1 万行日志找错误，不如让 hook grep 出 `ERROR` 只返回匹配行。） | 把"海量原始输出"变成"几行结论"。 |
| **代码智能插件（LSP）** | 官方推荐"跳转定义/查找引用"替代整棵树的 grep 扫描（见 `large-codebases` 与插件文档）。 | 减少探索性文件读取。 |
| **`/btw` 侧边提问** | 答案出现在可关闭的浮层里，**永不进入对话历史**——"you can check a detail without growing context."（查个细节而不撑大上下文。） | 小问题不污染主上下文。 |
| **`/rewind` / Esc+Esc 部分压缩** | 选中消息检查点后"Summarize from here / up to here"，只压缩一部分对话。 | 比整体 `/compact` 更精细。 |

---

## 七、几个容易被忽略的官方细节

### 7.1 长会话的 token 消耗是"每次请求全量重发"

`costs` 文档解释了为什么挂了一天的会话"看起来没干什么却很贵"：

> "Claude Code sends your full conversation with every request… so a one-line question in a session that has been open all day still draws usage for the whole conversation."
> （Claude Code 每次请求都发送你的完整对话……所以一个挂了一整天的会话里问一句一行的话，仍然消耗整个对话的量。）

这正是 prompt caching 存在的意义——重复内容按"缓存读"计费。而缓存被破坏（切模型、改 effort、开 fast mode、连/断 MCP、压缩、升级）的那一下，就是"慢 + 贵"的瞬间。

### 7.2 想省缓存命中率，就少做"中途变更"

`prompt-caching` 文档给了一条非常实用的操作建议：

> "Pick your model and effort level at the top of a session, then save `/compact` for natural breaks between tasks. The fewer changes you make mid-task, the higher your cache hit rate."
> （在会话开头就定好模型和 effort 级别，把 `/compact` 留到任务之间的自然停顿。任务中途改动越少，缓存命中率越高。）

### 7.3 会话中途改 CLAUDE.md 不生效

`prompt-caching` 文档：项目根和用户级 CLAUDE.md 只在会话启动时读一次、保存在内存里——

> "Editing them mid-session does not invalidate the cache, but the edit also doesn't apply. The new content loads on the next `/clear`, `/compact`, or restart."
> （会话中途编辑它们不会使缓存失效，但改动也不生效。新内容要等下次 `/clear`、`/compact` 或重启才加载。）

子目录嵌套 CLAUDE.md 和路径规则是在"首次读到匹配文件"时加载的，所以在加载前编辑仍然生效。

### 7.4 auto memory 是独立的第三个世界

`memory` 文档明确：auto memory（`~/.claude/projects/<project>/memory/`）**不加载进 subagent**，且与 CLAUDE.md 分属两套系统。它的加载上限独立于 CLAUDE.md：

> "The first 200 lines of `MEMORY.md`, or the first 25KB, whichever comes first, are loaded at the start of every conversation."
> （`MEMORY.md` 的前 200 行，或前 25KB，先到者为准，在每次对话启动时加载。）

---

## 八、结论：官方语境里的"上下文管理"是什么

把官方六份文档串起来，可以归纳成一句话的思维模型：

> **上下文 = 整个会话的完整记忆，是最贵的资源；管理 = 让"该进的东西"按需进入、"不该占的东西"及时退出。**

具体落地为三层：

1. **减少常驻**：CLAUDE.md 压在 200 行内，专业内容挪进 skill / 路径规则 / MCP 按需加载——"启动即占"的只有系统提示词、记忆和必要的项目约定。
2. **隔离大头**：探索、跑测试、读大文件这类"高占用"动作丢给 subagent 或 hook 预处理，让主窗口只收到摘要。
3. **适时重置**：任务切换用 `/clear`，长任务中途用 `/compact`（带焦点），压缩前把要长期保留的规则写进 CLAUDE.md——因为**自动压缩会丢掉早期对话的细节**，只有 CLAUDE.md 和 auto memory 会从磁盘重生。

---

## 参考来源

本文内容综合以下官方资料整理（**均于 2026-08-04 通过 web_fetch 获取**）：

- **How Claude Code works** — https://code.claude.com/docs/en/how-claude-code-works
  （上下文窗口定义、"When context fills up"自动压缩机制、subagent/skill 管理上下文的定位；"agentic harness / context management" 原话）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （交互式时间轴：启动加载项与示意 token、文件读取成本、subagent 的 6,100→420 token 节省量；"What survives compaction"存活表；手动 `/compact` / `/clear` / 1M 窗口；"Token counts are illustrative" 原话）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"The context window is the most important resource to manage"、"performance degrades as it fills" 等核心原话；Manage context aggressively 一节：`/clear`、`/compact <instructions>`、`/btw`、subagent 调查）
- **Manage costs effectively** — https://code.claude.com/docs/en/costs
  （"Token costs scale with context size"；Reduce token usage 一节的完整策略：CLI 工具 vs MCP、hooks 预处理、skill 卸载 CLAUDE.md；"为什么长会话耗 token"的机制解释）
- **How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （CLAUDE.md 与 auto memory 两套记忆系统的加载上限："前 200 行或 25KB"；CLAUDE.md 建议 200 行内；压缩后什么能恢复）
- **How Claude Code uses prompt caching** — https://code.claude.com/docs/en/prompt-caching
  （请求重发完整上下文的机制、缓存分层、使缓存失效的动作清单、"中途改 CLAUDE.md 不生效"、"Pick your model and effort level at the top of a session" 原话）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（写 CLAUDE.md 与提示词的配套实践）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（大代码库上下文组织：分层 CLAUDE.md + LSP + subagent）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
