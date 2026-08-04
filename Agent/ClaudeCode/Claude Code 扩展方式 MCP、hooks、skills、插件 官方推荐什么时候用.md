# Claude Code 扩展方式 MCP、hooks、skills、插件 官方推荐什么时候用？

> **一句话总结**：官方文档对 Claude Code 的扩展方式有一个很明确的分层——**skills、MCP、hooks 是三种"能力"，plugins 是"打包分发层"**：**skills 在"复用知识/可调用的工作流"时用，MCP 在"连外部服务/数据/动作"时用，hooks 在"某个动作必须每次都发生、且不能靠模型自觉"时用；而 plugin 的推荐时机只有一个——"同一套配置要在多个仓库复用，或要分发给团队/社区"。** 官方还专门给了一张"由触发信号决定加什么"的渐进式搭建表，本文一并翻译整理。
>
> 本文基于 Claude Code 官方文档 `Extend Claude Code`（官方专门回答"when to use CLAUDE.md, Skills, subagents, hooks, MCP, plugins"的专页）、`Extend Claude with skills`、`Connect Claude Code to tools via MCP`、`Automate actions with hooks`、`Create plugins`，以及官方 Claude 博客 `Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents` 整理，文末附参考来源。

很多人在深入用 Claude Code 一段时间后，会遇到同一个困惑：想自定义 Claude 的行为，到底用 MCP、hooks、skills 还是插件？网上的例子五花八门——有人用 hook 做 lint，有人用 skill 做 deploy，有人用 MCP 接数据库，还有人把三样都塞进插件。

直觉上，这四种方式像是"同一种东西的四个选项"。但官方文档给出的答案恰好相反：**它们不是并列的选项，而是分工不同的三层**——skills / MCP / hooks 各自解决一类能力问题，plugins 则是一个"打包盒"，把前三种（以及 subagent、LSP、monitors 等）装进去统一分发。

官方为此专门开了一页 `Extend Claude Code`，副标题就是"理解什么时候用 CLAUDE.md、Skills、subagents、hooks、MCP、plugins"。本文就围绕这一页，把四种方式的官方推荐场景拆开讲。

---

## 一、先搞清结构：三种"能力" + 一层"打包"

官方文档对扩展层的总览是这么说的（原文）：

> "Extensions plug into different parts of the agentic loop."（扩展插在 agentic loop 的不同环节上。）

具体到我们关心的四种：

| 扩展 | 官方一句话定位（原话） | 本质 |
|---|---|---|
| **Skills** | "add reusable knowledge and invocable workflows"（增加可复用的知识与可调用的工作流） | 能力：知识与流程 |
| **MCP** | "connects Claude to external services and tools"（把 Claude 连到外部服务与工具） | 能力：外部连接 |
| **Hooks** | "run your script, HTTP request, prompt, or subagent when Claude Code reaches a lifecycle event"（在 Claude Code 到达某个生命周期事件时运行你的脚本/HTTP 请求/提示词/subagent） | 能力：确定性自动化 |
| **Plugins** | "package and distribute these features"（打包并分发这些特性） | **打包层**：不是独立能力 |

关键就是最后一行。官方明确说：

> "[Plugins] are the packaging layer. A plugin bundles skills, hooks, subagents, and MCP servers into a single installable unit."（插件是打包层。一个插件把 skills、hooks、subagents 和 MCP servers 打包成一个可安装的单元。）

所以"插件 vs skill / MCP / hook"本身是个伪命题——插件能装下 skill、MCP server 和 hook，是一个"分发单元"，而不是第四个"能力类型"。`Create plugins` 文档的插件结构表也印证了这点：插件根目录下可以有 `skills/`、`agents/`、`hooks/`、`.mcp.json`、`.lsp.json`、`monitors/`、`settings.json`、`bin/`（往 Bash 工具的 PATH 里加可执行文件）。

## 二、核心对比：官方推荐什么时候用谁

官方专页给了一张主表（Feature → What it does → When to use it → Example）。截取我们关心的四行：

| 扩展 | What it does（做什么） | **When to use it（什么时候用）** | Example（例子） |
|---|---|---|---|
| **Skill** | Instructions, knowledge, and workflows Claude can use（可被 Claude 使用的指令/知识/工作流） | **Reusable content, reference docs, repeatable tasks**（可复用内容、参考文档、重复任务） | `/deploy` 跑部署清单；含 API 端点模式的 API 文档 skill |
| **MCP** | Connect to external services（连接外部服务） | **External data or actions**（外部数据或动作） | 查数据库、发 Slack、控制浏览器 |
| **Hook** | Script, HTTP request, prompt, or subagent triggered by events（由事件触发的脚本/请求/提示词/subagent） | **Automation that must run on every matching event**（必须在每个匹配事件上都运行的自动化） | 每次文件编辑后跑 ESLint |
| **Plugin** | （打包层）bundle 以上能力并命名空间化 | **Reuse the same setup across multiple repositories or distribute to others via a marketplace**（在多个仓库间复用同一套配置，或通过 marketplace 分发） | 团队共享的代码审查插件 |

官方原话对 plugin 时机说得更完整：

> "Use plugins when you want to reuse the same setup across multiple repositories or distribute to others via a **[marketplace]**."（当你想要在多个仓库间复用同一套配置，或通过 marketplace 分发给别人时，用插件。）

一句话记住这张表：**skill 管"知识/流程"，MCP 管"外面世界"，hook 管"必须发生的事"，plugin 管"打包带走"。**

## 三、逐项拆解官方原话

### 1. Skills：最灵活的扩展，管"可复用内容与可调用流程"

官方专页开头就给了个定性：

> "Skills are the most flexible extension. A skill is a markdown file containing knowledge, workflows, or instructions."（Skills 是最灵活的扩展。一个 skill 就是一份包含知识、工作流或指令的 markdown 文件。）

什么时候该建 skill？`Extend Claude with skills` 文档给了两条非常具体的触发信号：

> "Create a skill when you keep pasting the same instructions, checklist, or multi-step procedure into chat, or when a section of CLAUDE.md has grown into a procedure rather than a fact."（当你反复把同一段指令、清单或多步流程粘贴进对话，或 CLAUDE.md 的某一段已经"从事实长成了流程"时，就该建 skill。）

> "Unlike CLAUDE.md content, a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."（和 CLAUDE.md 不同，skill 的正文只有在被用到时才加载，所以长参考素材在你真正需要之前几乎不占上下文。）

官方博客 `Steering Claude Code` 说得更直接：

> "Instructions that are procedural, like deploy workflows, release checklists, or review processes, belong in a skill rather than in CLAUDE.md."（像部署工作流、发布清单、审查流程这类**过程性**指令，应该放 skill，而不是 CLAUDE.md。）

> "Procedures belong in skills."（流程属于 skill。）

一句话：**CLAUDE.md 放"事实"（build 命令、约定），skill 放"流程"（deploy、release、review）。** 判别标准是"加载时机"——skill 的正文按需加载，适合低频但完整的大段内容。

### 2. MCP：连"外部"，不解决"知识"与"确定性"

MCP 的官方文档 `Connect Claude Code to tools via MCP` 开头第一段给了核心判别信号：

> "Connect a server when you find yourself copying data into chat from another tool, like an issue tracker or a monitoring dashboard. Once connected, Claude can read and act on that system directly instead of working from what you paste."（当你发现自己总在把某个工具里的数据复制粘贴进对话——比如 issue 跟踪器、监控面板——就该连一个 MCP server。连上后，Claude 可以直接读并操作那个系统，而不是靠你粘贴。）

> "MCP servers give Claude Code access to your tools, databases, and APIs."（MCP servers 让 Claude Code 能访问你的工具、数据库和 API。）

官方列的典型用途很能说明"外部"这个定位：

- 从 issue 跟踪器实现功能（"Add the feature described in JIRA issue ENG-4521 and create a PR on GitHub"）
- 查监控数据（Sentry、Statsig）
- 查数据库（PostgreSQL）
- 对接设计稿（Figma / Slack）
- 自动化工作流（Gmail 草稿）
- 响应外部事件（MCP server 作 channel 推送消息进来）

官方专页还用一张对比表讲清了 **MCP vs Skill** 的分工——这是最容易混的一组：

| 维度 | MCP | Skill |
|---|---|---|
| **What it is** | Protocol for connecting to external services（连接外部服务的协议） | Knowledge, workflows, and reference material（知识、工作流、参考素材） |
| **Provides** | Tools and data access（工具与数据访问） | Knowledge, workflows, reference material（知识、工作流、参考素材） |
| **Examples** | Slack 集成、数据库查询、浏览器控制 | 代码审查清单、deploy 工作流、API 风格指南 |

官方原话的结论是：**两者解决不同问题，而且常常配合使用**——

> "MCP gives Claude purpose-built tools for an external system, with the connection and authentication handled by the server."（MCP 给 Claude 提供针对某个外部系统的专用工具，连接与认证由 server 处理。）

> "Skills give Claude knowledge about how to use those tools effectively, plus workflows you can trigger with `/<name>`."（Skills 给 Claude 关于**如何用好那些工具**的知识，以及可以用 `/<name>` 触发的工作流。）

例：**MCP server 连数据库；skill 教 Claude 你的数据模型、常见查询模式、不同任务该用哪张表。** 这就是官方给的"Skill + MCP"组合模式。

### 3. Hooks：确定性自动化与护栏，不靠模型自觉

hooks 是四种里最"硬"的一种。官方文档 `Automate actions with hooks` 的开篇原话：

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you **deterministic control**: certain actions always happen rather than relying on the LLM to choose to run them. Use hooks to enforce project rules, automate repetitive tasks, and integrate Claude Code with your existing tools."（Hooks 是用户自定义的 shell 命令。Claude Code 在生命周期的特定节点运行它们，给你**确定性的控制**：某些动作**总是会发生**，而不是依赖 LLM 选择去跑。用 hooks 来强制执行项目规则、自动化重复任务、把 Claude Code 接进现有工具链。）

注意"deterministic control（确定性控制）"这个词——这是 hooks 和其他方式最根本的分野。官方专页的 **Hook vs Skill** 对比表把这条讲透了：

| 维度 | Hook | Skill |
|---|---|---|
| **Runs** | A shell command, HTTP request, LLM prompt, or subagent（一条 shell 命令 / HTTP 请求 / LLM 提示词 / subagent） | Instructions Claude reads and follows（Claude 阅读并遵循的指令） |
| **Triggered by** | Lifecycle events（如 `PostToolUse`、`SessionStart`） | 你输入 `/<name>`，或 Claude 根据描述匹配任务 |
| **Determinism** | **Always fires on its event; the trigger is guaranteed**（事件一到必触发，触发器是保证的） | Claude 解读指令，结果可能因人而异 |
| **Context cost** | **Zero** unless the hook returns output（零，除非 hook 返回输出） | Description 每次会话加载，全文用到才加载 |
| **Best for** | Linting after edits, blocking unsafe commands, logging, notifications | 需要推理的工作流、参考素材、多步任务 |

官方原话给了两条非常干脆的决策规则：

> "Use a hook when the action must happen the same way every time and doesn't need Claude to think."（当某个动作每次都必须以同样的方式发生、且不需要 Claude 思考时，用 hook。）

> "Use a skill when Claude should decide how to apply the steps, or when the content is knowledge rather than a script."（当应该由 Claude 决定如何执行步骤，或内容是知识而非脚本时，用 skill。）

官方博客把"确定性"的深层原因讲得更狠——**"指令"不是"保证"**：

> "When there's something that absolutely must not happen, an instruction is the wrong tool."（当有些事情**绝对**不能发生时，一条指令是错误工具。）

> "A real guardrail needs to be deterministic, and the enforcement methods are hooks and permissions."（真正的护栏必须是确定性的，而落地的办法是 hooks 和 permissions。）

> "The model choosing to run a formatter is different from the formatter running automatically."（模型**选择**去跑格式化器，和格式化器**自动**跑起来，是两码事。）

一个官方反复强调的护栏场景：你想保证"绝不编辑 `.env`"——在 CLAUDE.md / skill 里写这句话只是"请求"，不是"保证"；用 `PreToolUse` hook 拦下这个编辑才是"执行"。hook 对 `PreToolUse` 返回 exit code 2 即拒绝工具调用（deny），而且**在 `dontAsk` 甚至 `bypassPermissions` 模式下仍会生效**——这是用户改权限模式也绕不过去的强制策略。

### 4. Plugins：不是"第四个能力"，是"分发单元"

前面已引过官方原话——plugins 是 packaging layer。`Create plugins` 文档给了更细的选型：**同一份自定义配置，什么时候放在 `.claude/` 裸配，什么时候打包成插件？**

| 方式 | Skill 命名 | **Best for（最适合）** |
|---|---|---|
| **Standalone**（`.claude/` 目录） | `/hello` | Personal workflows, project-specific customizations, quick experiments（个人工作流、单项目定制、快速试验） |
| **Plugins**（独立目录 + `plugin.json` 清单） | `/plugin-name:hello` | Sharing with teammates, distributing to community, versioned releases, reusable across projects（分享给队友、分发给社区、带版本号的发布、跨项目复用） |

官方列出的"用插件"的完整条件：

> "Use plugins when:
> * You want to share functionality with your team or community（想分享给团队或社区）
> * You need the same skills/agents across multiple projects（多个项目需要同一套 skills/agents）
> * You want version control and easy updates for your extensions（想要版本管理与方便更新）
> * You're distributing through a marketplace（要通过 marketplace 分发）
> * You're okay with namespaced skills like `/my-plugin:hello`（能接受 `/my-plugin:hello` 这种带命名空间的命名）"

一个非常实用的官方建议——**先裸配，后打包**：

> "Start with standalone configuration in `.claude/` for quick iteration, then convert to a plugin when you're ready to share."（先在 `.claude/` 里用裸配置快速迭代，等准备好分享了再转成插件。）

> "The same triggers tell you when to update what you already have."（同样的触发信号也告诉你什么时候该升级你已有的配置。）

## 四、官方"渐进式搭建"表：由触发信号决定加什么

官方专页强调"你不需要一开始就配齐所有东西"（"You don't need to configure everything up front"），并给了最实用的一张表——**每个特征都有可识别的触发信号，多数团队大致按这个顺序加**：

| 触发信号（官方原话） | 加什么 |
|---|---|
| "Claude gets a convention or command wrong twice"（Claude 把某个约定/命令搞错两次） | 写进 **CLAUDE.md** |
| "You keep typing the same prompt to start a task"（你反复打同一句提示词来开任务） | 存成用户可调用的 **skill** |
| "You paste the same playbook or multi-step procedure into chat for the third time"（同一个手册/多步流程第三次粘贴进对话） | 捕获成 **skill** |
| "You keep copying data from a browser tab Claude can't see"（你反复从 Claude 看不到的浏览器标签页复制数据） | 连成 **MCP server** |
| "Claude reads many files to find where a symbol is defined or used"（Claude 读很多文件才能找到符号定义/使用处） | 装**代码智能插件**（LSP） |
| "A side task floods your conversation with output you won't reference again"（旁路任务刷屏主对话、之后又用不到） | 走 **subagent** |
| "You want something to happen every time without asking"（你想让某件事每次都发生、不用开口） | 写一个 **hook** |
| "A second repository needs the same setup"（第二个仓库也需要同一套配置） | 打包成 **plugin** |

这张表最值得记的是**顺序**：先 CLAUDE.md（约定）→ 再 skill（重复流程）→ 然后 MCP（外部连接）→ 再代码智能/ subagent（性能）→ hook（确定性）→ 最后 plugin（跨仓库复用）。它把"什么时候用哪种"从抽象原则变成了可对号入座的信号。

## 五、串起来的决策路径

把官方各处原话整理成一张可执行的"选型口诀"：

| 你的症状 | 官方推荐 | 依据 |
|---|---|---|
| "每次都手动贴同一段流程/清单" | **Skill** | "Create a skill when you keep pasting the same instructions…" |
| "要从另一个工具/网站复制数据给 Claude" | **MCP** | "Connect a server when you find yourself copying data into chat from another tool…" |
| "想让某件事每次都自动发生、不能靠它自觉" | **Hook** | "Use hooks for anything that should happen deterministically…" |
| "这条规则**绝不允许**被违反（护栏）" | **Hook**（PreToolUse 拦截） | "A real guardrail needs to be deterministic…" |
| "同一套配置要在第二个仓库/团队里复用" | **Plugin** | "Use plugins when you want to reuse the same setup across multiple repositories…" |
| "只想自己快速试试" | **裸配 `.claude/`**，先别打包 | "Start with standalone configuration… then convert to a plugin when you're ready to share" |
| "既要连数据库，又要 Claude 知道怎么查" | **Skill + MCP** 组合 | "MCP provides the connection; a skill teaches Claude how to use it well" |
| "deploy 前要 lint 一下，不 lint 就不许过" | **Hook**（跑 lint + 拦非 lint 结果） | "Best for: Linting after edits, blocking unsafe commands" |

最后三个容易踩的坑，单独强调：

- **别用 MCP 去解决"知识"问题。** MCP 管"连接与数据访问"，skill 管"怎么用好"。想给 Claude 塞领域知识，答案是 skill，不是再挂一个 MCP server。
- **别用 skill 去"保证"某事不发生。** 指令只是请求，不是保证（"an instruction is the wrong tool"）。要强制，用 hook / permissions。
- **别一上来就写插件。** 官方明说先 `.claude/` 裸配快速迭代，等要分享/跨仓库了再转插件——插件多了一层命名空间（`/my-plugin:hello`）和版本管理，单独开发时是纯负担。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 通过 web_fetch 获取）：

- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （官方回答"when to use CLAUDE.md, Skills, subagents, hooks, MCP, plugins"的专页：扩展层总览、"插件是打包层"的定位、Feature→When to use 主表、Skill vs Subagent / CLAUDE.md vs Skill / MCP vs Skill / Hook vs Skill 对比表、"渐进式搭建"触发信号表、各特性上下文成本表）
- **Extend Claude with skills** — https://code.claude.com/docs/en/skills
  （"Create a skill when you keep pasting the same instructions…"原话、skill 正文按需加载、custom commands 并入 skills、frontmatter 的 `description`/`when_to_use` 字段）
- **Connect Claude Code to tools via MCP** — https://code.claude.com/docs/en/mcp
  （"Connect a server when you find yourself copying data into chat from another tool…"原话、MCP 典型用途清单、tool search、认证/作用域）
- **Automate actions with hooks** — https://code.claude.com/docs/en/hooks-guide
  （"deterministic control"原话、"Use hooks to enforce project rules…"、prompt-based/agent-based hooks、PreToolUse exit code 2 deny、hooks 与权限模式的关系）
- **Create plugins** — https://code.claude.com/docs/en/plugins
  （Standalone vs Plugins 选型表、"Use plugins when…"条件清单、"Start with standalone… then convert to a plugin"、插件结构：skills/agents/hooks/.mcp.json/.lsp.json/monitors/bin）
- **Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents（官方博客）** — https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more
  （"Each method trades context cost against authority"框架、"Procedures belong in skills"、"Use hooks for anything that should happen deterministically"、"A real guardrail needs to be deterministic"、"an instruction is the wrong tool"、插件作为 bundling mechanism）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 与上下隔离）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（LSP 代码智能插件与 MCP 接入的官方定位）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md 管理）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
