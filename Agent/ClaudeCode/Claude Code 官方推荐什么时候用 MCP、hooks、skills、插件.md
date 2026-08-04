# Claude Code 官方推荐什么时候用 MCP、hooks、skills、插件？

> **一句话总结**：四种扩展方式在官方文档里**不是四个平级选项，而是"三兄弟 + 一个打包层"**——**MCP 连外部系统**（有外部数据/动作要接、Claude 够不着时用）、**hooks 做确定性自动化与护栏**（某事必须每次发生时用）、**skills 装可复用知识与工作流**（你反复粘贴同一套东西、CLAUDE.md 里某段从"事实"长成"流程"时用）；而**插件是打包层（packaging layer）**，把前三者（外加 subagents、LSP、monitors）捆成一个可安装单元，**需要跨项目复用或分发时**才用。官方最强调的一条分界：**"每次都要发生、且不需要 Claude 思考"→ 用 hook；"该由 Claude 决定怎么执行"→ 用 skill**。
>
> 本文基于 Claude Code 官方文档 `Extend Claude Code`（features-overview）、`Extend Claude with skills`、`Connect Claude Code to tools via MCP`、`Automate actions with hooks`、`Create plugins`，以及官方 Claude 博客 `Steering Claude Code` 整理，所有引用均经 web_fetch 抓取原文核对，文末附参考来源。

用 Claude Code 一段时间后，几乎每个人都会遇到同一个选择题：想让它"更厉害"，到底是配一个 MCP server、写个 hook、建个 skill，还是干脆装个插件？

这四个词在社区里经常被并列讨论，但官方文档对它们的定位**根本不在一个维度上**。本文完全以官方文档为准，把"每种扩展是什么、官方建议什么时候用"逐条理清。

---

## 一、先看官方的总览：四种扩展不是平级选项，而是"三兄弟 + 一个打包层"

`Extend Claude Code`（`features-overview`）是官方专门回答"when to use CLAUDE.md, Skills, subagents, hooks, MCP, and plugins"的专页。它对扩展层的定义是：

> "This guide covers the extension layer: features you add to customize what Claude knows, connect it to external services, and automate workflows."
> （本指南覆盖**扩展层**：你添加的、用来定制 Claude 知道什么、连接外部服务、以及自动化工作流的那些功能。）

页面开头的总览先给了一句统领全局的话，点明这些扩展不是散装功能，而是插在 agent 循环的不同环节上：

> "Extensions plug into different parts of the agentic loop."
> （扩展插在 **agentic loop 的不同环节**上。）

随后四种扩展各占一句话，定位差异一目了然：

| 扩展 | 官方一句话定位（原话） | 本质 |
|---|---|---|
| **Skills** | "add reusable knowledge and invocable workflows"（增加**可复用的知识**与**可调用的工作流**） | 能力：知识与流程 |
| **MCP** | "connects Claude to external services and tools"（把 Claude **连到外部服务与工具**） | 能力：外部连接 |
| **Hooks** | "run your script, HTTP request, prompt, or subagent when Claude Code reaches a lifecycle event"（在 Claude Code 到达某个**生命周期事件**时运行你的脚本/HTTP 请求/提示词/subagent） | 能力：确定性自动化 |
| **Plugins** | "package and distribute these features"（**打包并分发**上面这些功能） | **打包层**：不是独立能力 |

页面的核心是一张"把你的目标匹配到功能"的对照表（Match features to your goal），四种扩展在这张表里的定位是：

| 扩展 | 它做什么 | 什么时候用它（官方原话） | 官方示例 |
|---|---|---|---|
| **Skill** | Instructions, knowledge, and workflows Claude can use | Reusable content, reference docs, repeatable tasks（可复用内容、参考文档、重复任务） | `/deploy` runs your deployment checklist（`/deploy` 跑部署清单） |
| **MCP** | Connect to external services（连接外部服务） | External data or actions（外部数据或动作） | Query your database, post to Slack, control a browser（查库、发 Slack、控浏览器） |
| **Hook** | Script, HTTP request, prompt, or subagent triggered by events | Automation that must run on every matching event（必须在每个匹配事件上都运行的自动化） | Run ESLint after every file edit（每次编辑后跑 ESLint） |
| **Plugins** | （打包层，表格下方单独说明） | 跨仓库复用 / 通过 marketplace 分发 | 把 skills、hooks、MCP 打包成一个 `/my-plugin:xxx` 单元 |

关键点在于：官方对插件的定位**不在表格里跟三者并列**，而是单独一句话：

> "[Plugins] are the packaging layer. A plugin bundles skills, hooks, subagents, and MCP servers into a single installable unit. Plugin skills are namespaced (like `/my-plugin:review`) so multiple plugins can coexist. Use plugins when you want to reuse the same setup across multiple repositories or distribute to others via a marketplace."
> （插件是**打包层**。一个插件把 skills、hooks、subagents 和 MCP servers 打包成一个**可安装单元**；插件里的 skill 会带 `/my-plugin:review` 这样的命名空间，让多个插件可以共存。当你想**跨多个仓库复用同一套配置**、或通过 marketplace **分发给别人**时，用插件。）

所以先记住这个总框架：**MCP / hooks / skills 是三种"解决不同问题"的机制，插件是装它们三个（以及 subagent、LSP、monitors 等）的盒子。** 下面分别展开。

---

## 二、MCP：需要"连接外部系统"时用

官方对 MCP 的定位非常一致，就是"连外部"：

> "MCP servers give Claude Code access to your tools, databases, and APIs."
> （MCP server 让 Claude Code 能访问你的工具、数据库和 API。）

它回答的**不是**"怎么让 Claude 更聪明"，而是"怎么让 Claude 够得着我这个系统"。官方 `mcp` 文档给了最直接的判断信号：

> "Connect a server when you find yourself copying data into chat from another tool, like an issue tracker or a monitoring dashboard. Once connected, Claude can read and act on that system directly instead of working from what you paste."
> （**当你发现自己总在把另一个工具的数据复制粘贴进对话时**——比如 issue 追踪器或监控面板——就连接一个 server。连上之后，Claude 可以直接读写那个系统，而不是靠你粘贴的内容干活。）

换言之，**"从某个工具复制数据进对话"这个动作，就是官方给的 MCP 触发器**。复制意味着两件事：一、那个系统 Claude 看不到；二、粘贴的过程既慢又丢信息。MCP 把"粘贴"升级成"直接读 + 直接操作"。

`features-overview` 的"逐步搭建"表里，对应的触发条件表达得更口语化：

| 触发信号 | 官方建议 |
|---|---|
| You keep copying data from a browser tab Claude can't see（你老是从 Claude 看不见的浏览器标签页里复制数据） | Connect that system as an MCP server（把这个系统连成 MCP server） |

官方 `mcp` 文档列了六类典型用途，共同点是**外部系统 + 数据/动作**：

- **从 issue 追踪器实现功能**："Add the feature described in JIRA issue ENG-4521 and create a PR on GitHub."
- **查监控数据**："Check Sentry and Statsig to check the usage of the feature described in ENG-4521."
- **查数据库**："Find emails of 10 random users who used feature ENG-4521, based on our PostgreSQL database."
- **对接设计稿**：按 Slack 里新发的 Figma 设计更新标准邮件模板。
- **自动化工作流**："Create Gmail drafts inviting these 10 users to a feedback session…"
- **响应外部事件**：MCP server 还可以作为 channel 把消息推进会话，让你离开时 Claude 也能响应 Telegram、Discord 或 webhook 事件。

一句话记忆：**"Claude 够不着的东西，要么你复制粘贴，要么接 MCP。"** 上下文成本上，MCP 的 tool 名称在会话启动时加载、完整 schema 用到才加载，空闲时占用很低（详见第七节）。

---

## 三、Hooks：需要"确定性自动化 / 护栏"时用

hooks 和 skills 是官方文档里专门做了"什么时候用哪个"对比的一对，核心差异是**确定性**。`Automate actions with hooks` 开篇就说：

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you deterministic control: certain actions always happen rather than relying on the LLM to choose to run them. Use hooks to enforce project rules, automate repetitive tasks, and integrate Claude Code with your existing tools."
> （Hooks 是用户定义的 shell 命令，Claude Code 在生命周期特定节点运行它们，从而给你**确定性控制**：某些动作**总会发生**，而不是依赖 LLM 自己选择去运行。用 hooks 来**强制执行项目规则**、自动化重复任务、把 Claude Code 接入你现有的工具。）

注意这句话反复出现的词：**deterministic（确定性）、always happen（总会发生）、rather than relying on the LLM to choose（而不是依赖 LLM 选择）**。这是 hooks 区别于 skills 的根本——hooks 不是"建议 Claude 做"，而是"系统替你强制执行"。

`features-overview` 里的触发条件最直白：

| 触发信号 | 官方建议 |
|---|---|
| You want something to happen every time without asking（你想让某事**每次自动发生、不用开口**） | Write a hook（写一个 hook） |

官方给的"hook vs skill"分界原话是：

> "Use a hook when the action must happen the same way every time and doesn't need Claude to think."
> （当动作必须**每次以完全相同的方式发生、且不需要 Claude 思考**时，用 hook。）

> "Use a skill when Claude should decide how to apply the steps, or when the content is knowledge rather than a script."
> （当应该由 **Claude 决定如何执行**这些步骤，或者内容是知识而非脚本时，用 skill。）

hooks 最容易被忽略的价值是**护栏（guardrail）**。官方原文一针见血：

> "Put guardrails in hooks. An instruction like 'never edit `.env`' in CLAUDE.md or a skill is a request, not a guarantee. A `PreToolUse` hook that blocks the edit is enforcement. If a rule must hold every time, make it a hook rather than a prompt instruction."
> （**把护栏放在 hooks 里。** CLAUDE.md 或 skill 里"永远别改 `.env`"这种指令是**请求**，不是**保证**；一个拦截该编辑的 `PreToolUse` hook 才是**强制执行**。如果某条规则必须每次成立，就做成 hook，而不是一句提示词指令。）

这个护栏的强度是有实锤的：`PreToolUse` hook 返回 **exit code 2** 即拒绝该工具调用（官方 `hooks` 参考："Exit 2 means a blocking error… `PreToolUse` blocks the tool call"），也可以用 JSON 输出返回 `permissionDecision: "deny"` 直接拦下。官方 `permissions` 文档还确认了执行时机——**PreToolUse hooks 在权限弹窗之前运行**（"When Claude Code makes a tool call, PreToolUse hooks run before the permission prompt"），所以它是比提示词更靠前的强制层。同时官方划了一条边界：**hook 的决策不能绕过权限规则**（"Hook decisions don't bypass permission rules"，deny/ask 规则照样生效）——因此护栏的完整图景是官方博客那句原话：真正的护栏必须是确定性的，而强制执行的手段是 **hooks 和 permissions 两者合起来**。

配套的官方博客《Steering Claude Code》把"指令 vs 保证"的道理说得更透：

> "When there's something that absolutely must not happen, an instruction is the wrong tool."
> （当有些事情**绝对**不能发生时，一条指令是错误工具。）

> "A real guardrail needs to be deterministic, and the enforcement methods are hooks and permissions."
> （真正的护栏必须是确定性的，而强制执行的手段就是 hooks 和权限。）

> "The model choosing to run a formatter is different from the formatter running automatically."
> （模型**选择**去跑格式化工具，和格式化工具**自动运行**，是两码事。）

hooks 也不只限于 shell 命令——`hooks-guide` 里列了 command / HTTP / prompt / agent 四种（`hooks` 参考里还有 `mcp_tool` 型：调用一个已连接的 MCP server 上的工具）。其中"需要判断力而非确定性规则"的决策，可以用 prompt 型或 agent 型 hook 让模型去评估：

> "For decisions that require judgment rather than deterministic rules, you can also use prompt-based hooks or agent-based hooks that use a Claude model to evaluate conditions."
> （对需要**判断**而非确定性规则的决定，你也可以用 prompt-based 或 agent-based hooks，让一个 Claude 模型来评估条件。）

一句话记忆：**"每次都要发生、不需要动脑子 → hook；需要判断 → skill 或 prompt/agent 型 hook。"**

---

## 四、Skills：需要"可复用知识 / 工作流"时用

skills 是官方文档反复强调的**最灵活的扩展**：

> "Skills are the most flexible extension. A skill is a markdown file containing knowledge, workflows, or instructions. You can invoke skills with a command like `/deploy`, or Claude can load them automatically when relevant."
> （Skills 是**最灵活**的扩展。一个 skill 就是一份包含知识、工作流或指令的 markdown 文件。你可以用 `/deploy` 这样的命令调用它，也可以让 Claude 在相关时自动加载它。）

官方 `skills` 文档给的使用信号非常具体：

> "Create a skill when you keep pasting the same instructions, checklist, or multi-step procedure into chat, or when a section of CLAUDE.md has grown into a procedure rather than a fact."
> （当你**反复往对话里粘贴**同样的指令、清单或多步流程，或者 CLAUDE.md 里某段已经从"事实"长成了"流程"时，就创建一个 skill。）

`features-overview` 的"逐步搭建"表给了两条几乎同义的条件：

| 触发信号 | 官方建议 |
|---|---|
| You keep typing the same prompt to start a task（你老是用同一句 prompt 起一个任务） | Save it as a user-invocable skill（存成用户可调用的 skill） |
| You paste the same playbook or multi-step procedure into chat for the third time（同一份 playbook 或流程粘贴到聊天里**第三次**） | Capture it as a skill（把它固化成 skill） |

skill 和 CLAUDE.md 的关键区别是**加载时机**，这决定了它的成本特性：

> "Unlike CLAUDE.md content, a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."
> （和 CLAUDE.md 不同，skill 的正文**只在被使用时才加载**，所以长篇参考材料在用到之前几乎零成本。）

官方博客《Steering Claude Code》对"CLAUDE.md vs skill"给了一条更简洁的经验法则：

> "CLAUDE.md is for facts Claude should hold all the time" / "Procedures belong in skills."
> （CLAUDE.md 放 Claude 应该**时刻记住的事实**；**流程属于 skill**。）

> "A deployment runbook or a security review checklist should live in `.claude/skills/`"
> （部署手册或安全审查清单应该放在 `.claude/skills/` 里。）

官方还把 skill 分成两类：

> "Skills can be reference or action. Reference skills provide knowledge Claude uses throughout your session (like your API style guide). Action skills tell Claude to do something specific (like `/deploy` that runs your deployment workflow)."
> （Skill 分**参考型**和**动作型**。参考型提供 Claude 整个会话中都会用到的知识（比如你的 API 风格指南）；动作型告诉 Claude 做某件具体的事（比如运行部署工作流的 `/deploy`）。）

注意一个重要的演进：**自定义命令（custom commands）已经合并进 skills**。`.claude/commands/deploy.md` 和 `.claude/skills/deploy/SKILL.md` 都生成 `/deploy` 且行为一致，官方推荐用 skills（支持目录、frontmatter 控制调用方、自动加载等额外能力）。

一句话记忆：**"同一套知识/流程要反复用 → skill；且它只在用到时才占上下文。"**

---

## 五、插件：需要"打包分发 / 跨项目复用"时用

插件是四种里定位最特殊的一个——它不是和前三者并列的"第四种能力"，而是**装前三种的容器**。`Create plugins` 开篇：

> "Plugins let you extend Claude Code with custom functionality that can be shared across projects and teams."
> （插件让你用**可在项目间和团队间共享**的自定义功能扩展 Claude Code。）

一个插件能装的东西，`plugins` 文档有张目录表：`skills/`、`commands/`、`agents/`、`hooks/`、`.mcp.json`（MCP server 配置）、`.lsp.json`（LSP 代码智能）、`monitors/`（后台监控）、`bin/`（加入 Bash 工具 PATH 的可执行文件）、`settings.json`（默认设置）。

官方对"什么时候该从独立配置升级成插件"给出了明确分界，`plugins` 文档里有一整张对照表：

| 维度 | **Standalone**（`.claude/` 目录） | **Plugins**（插件目录） |
|---|---|---|
| Skill 命名 | `/hello` | `/plugin-name:hello`（带命名空间） |
| 最适合 | Personal workflows, project-specific customizations, quick experiments（个人工作流、单项目定制、快速试验） | Sharing with teammates, distributing to community, versioned releases, reusable across projects（团队共享、社区分发、版本化发布、跨项目复用） |

`features-overview` 的触发条件只有一条：

| 触发信号 | 官方建议 |
|---|---|
| A second repository needs the same setup（**第二个仓库**需要同一套配置） | Package it as a plugin（打包成插件） |

`plugins` 文档里"Use plugins when"的完整清单：

> "Use plugins when:
> * You want to share functionality with your team or community
> * You need the same skills/agents across multiple projects
> * You want version control and easy updates for your extensions
> * You're distributing through a marketplace
> * You're okay with namespaced skills like `/my-plugin:hello` (namespacing prevents conflicts between plugins)"
> （这些情况用插件：想和团队/社区共享功能；多个项目需要同样的 skills/agents；想要版本控制和便捷更新；要通过 marketplace 分发；能接受 `/my-plugin:hello` 这种命名空间化的 skill 名——命名空间正是为了**防止插件间冲突**。）

官方给的最务实路径是：

> "Start with standalone configuration in `.claude/` for quick iteration, then convert to a plugin when you're ready to share."
> （先在 `.claude/` 里用独立配置快速迭代，等**准备好分享时**再转成插件。）

官方博客《Steering Claude Code》的原话也印证了它是"收集再分发"的层：把 skills、subagents、hooks、输出风格等"bundle 成一个插件，在队友或项目间共享一套自洽的配置"。

一句话记忆：**"第一个项目里先用独立配置，第二个项目需要同一套时才打包成插件。"**

---

## 六、最容易混的两组对比：MCP vs Skill、Hook vs Skill

`features-overview` 专门设了"Compare similar features"区块，其中两组对比恰好是社区最常问的。

### 6.1 MCP vs Skill：一个"连得上"，一个"用得好"

| 维度 | **MCP** | **Skill** |
|---|---|---|
| What it is（是什么） | Protocol for connecting to external services（连接外部服务的协议） | Knowledge, workflows, and reference material（知识、工作流、参考材料） |
| Provides（提供什么） | Tools and data access（工具与数据访问） | Knowledge, workflows, reference material |
| Examples（示例） | Slack integration, database queries, browser control | Code review checklist, deploy workflow, API style guide |

官方明确指出两者**解决不同问题、且天然互补**：

> "MCP gives Claude purpose-built tools for an external system, with the connection and authentication handled by the server. Skills give Claude knowledge about how to use those tools effectively, plus workflows you can trigger with `/<name>`."
> （MCP 给 Claude 提供针对某个外部系统的**专用工具**，连接与认证由 server 处理；Skills 给 Claude **如何用好这些工具**的知识，以及用 `/<name>` 触发的工作流。）

> "Example: An MCP server connects Claude to your database. A skill teaches Claude your data model, common query patterns, and which tables to use for different tasks."
> （示例：MCP server 把 Claude 连到你的数据库；skill 教 Claude 你的数据模型、常用查询模式、以及不同任务该用哪些表。）

### 6.2 Hook vs Skill：一个"不用想"，一个"要想"

| 维度 | **Hook** | **Skill** |
|---|---|---|
| Runs（运行什么） | A shell command, HTTP request, LLM prompt, or subagent（一条 shell 命令 / HTTP 请求 / LLM 提示词 / subagent） | Instructions Claude reads and follows（Claude 读并遵循的指令） |
| Triggered by（谁触发） | Lifecycle events such as `PostToolUse` or `SessionStart`（生命周期事件，事件发生必触发） | You typing `/<name>`, or Claude matching the description（你敲 `/<name>`，或 Claude 匹配描述） |
| Determinism（确定性） | Always fires on its event; the trigger is guaranteed（事件一到必然触发） | Claude interprets the instructions; outcome can vary（Claude 解读指令，结果可能不同） |
| Context cost（上下文成本） | Zero unless the hook returns output（零，除非 hook 返回输出） | Description loads each session; full content loads when used |
| Best for（最适合） | Linting after edits, blocking unsafe commands, logging, notifications | Workflows that need reasoning, reference material, multi-step tasks |

官方的判定口诀就两行：

> "**Use a hook** when the action must happen the same way every time and doesn't need Claude to think. For example: format on save, reject `rm -rf /`, post a Slack message when a session ends."
> （动作必须每次一模一样、且不需要 Claude 思考 → **用 hook**。例如：保存即格式化、拒绝 `rm -rf /`、会话结束时发 Slack。）

> "**Use a skill** when Claude should decide how to apply the steps, or when the content is knowledge rather than a script. For example: a `/release` checklist, your API style guide, a debugging playbook."
> （应由 Claude 决定怎么执行、或内容是知识而非脚本 → **用 skill**。例如：`/release` 清单、API 风格指南、调试手册。）

另外值得注意 hook 输出与 skill 的分工：

> "Hook output lands in context. A `PostToolUse` hook that runs your linter feeds results back as text Claude reads; a `/fix-lint` skill tells Claude how to resolve them."
> （hook 的输出会进入上下文。`PostToolUse` hook 跑完 linter 会把结果作为文本喂给 Claude 读；而 `/fix-lint` skill 则是告诉 Claude 怎么修复这些 lint 问题。）

### 6.3 组合不是二选一

`features-overview` 的"Combine features"表给出几种官方认证的组合套路：

| 组合模式 | 怎么工作 | 示例 |
|---|---|---|
| **Skill + MCP** | MCP 提供连接，skill 教 Claude 怎么用好它 | MCP 连数据库，skill 记录 schema 与查询模式 |
| **Hook + MCP** | hook 通过 MCP 触发外部动作 | 编辑关键文件后，hook 发 Slack 通知 |

更进阶的组合：**skills 和 hooks 还是"设计 agent 循环"的积木**（官方博客原话："Skills and hooks are also the building blocks of designing agent loops."）。

---

## 七、上下文成本对照：每种扩展什么时候加载、花多少

`features-overview` 专门有一节讲"每种功能什么时候加载、上下文成本多少"，这对选择很有参考价值：

| 扩展 | 什么时候加载 | 加载什么 | 上下文成本 |
|---|---|---|---|
| **CLAUDE.md** | 会话启动 | 全部内容 | 每个请求都占 |
| **Skills** | 会话启动 + 使用时 | 启动时只加载描述，用到时才加载全文 | 低（每个请求占一条描述） |
| **MCP servers** | 会话启动 | 只加载工具名，完整 schema 按需加载 | 低，直到真正用某个工具 |
| **Hooks** | 触发时 | 什么都不加载（在外部运行） | **零**，除非 hook 返回内容 |

官方给的一句话建议：

> "Hooks are ideal for side effects (linting, logging) that don't need to affect Claude's context."
> （Hooks 非常适合那些**不需要影响 Claude 上下文**的副作用——linting、日志。）

以及一个可操作的技巧：如果你写了个只给自己用的 skill，可以在 frontmatter 里加 `disable-model-invocation: true`，让它的上下文成本降到零（不加载描述），只有你手动调用才生效。

---

## 八、结论：一张决策表 + 渐进式搭建表 + 三个坑

### 8.1 把官方所有表述收敛成"什么时候用哪个"的决策表

| 场景信号（官方原话） | 用什么 | 一句话理由 |
|---|---|---|
| 老从另一个工具复制数据进对话（"copying data into chat from another tool"） | **MCP** | Claude 够不着的外部数据/动作，用协议连上 |
| 想让它干个外部动作（查库、发 Slack、控浏览器） | **MCP** | 工具 + 连接 + 认证，server 全包 |
| 某事每次都要发生、不用问（"every time without asking"） | **Hook** | 确定性控制，事件触发即执行 |
| 某条规则必须每次成立（如"禁止改 .env"） | **Hook** | 指令是请求，hook 才是强制执行 |
| 反复粘贴同一段指令/清单/流程（"pasting the same instructions"） | **Skill** | 可复用知识/工作流，用到才加载 |
| 该由 Claude 判断怎么执行步骤 | **Skill** | skill 把执行方式交给模型 |
| 第二个仓库需要同一套配置（"a second repository needs the same setup"） | **Plugin** | 打包层：把 skills/hooks/agents/MCP 打包分发 |

回到开头的框架，用一句话总结四种扩展的官方定位差异：

> **MCP 管"够得着"（外部连通）、hooks 管"必然发生"（确定性自动化）、skills 管"反复使用"（可复用知识与流程）——三者解决不同问题；插件管"打包分发"，把前三种装进一个可共享的单元。**

### 8.2 官方"渐进式搭建"触发信号表（最实用的一张表）

`features-overview` 里还有一张按"症状"而不是按"功能"排的表——"Build your setup over time"（逐步搭建你的配置）。官方明确说 "You don't need to configure everything up front"（你不需要一开始就配齐所有东西），每个特征都有可识别的触发信号，多数团队大致按这个顺序加：

| 触发信号（官方原话） | 加什么 |
|---|---|
| Claude gets a convention or command wrong twice（Claude 把某个约定/命令搞错两次） | Add it to CLAUDE.md |
| You keep typing the same prompt to start a task（你反复打同一句提示词来开任务） | Save it as a user-invocable **skill** |
| You paste the same playbook or multi-step procedure into chat for the third time（同一份手册/多步流程第三次粘贴进对话） | Capture it as a **skill** |
| You keep copying data from a browser tab Claude can't see（你反复从 Claude 看不到的浏览器标签页复制数据） | Connect that system as an **MCP server** |
| Claude reads many files to find where a symbol is defined or used（Claude 读很多文件才能找到符号定义/使用处） | Install a code intelligence **plugin**（LSP） |
| A side task floods your conversation with output you won't reference again（旁路任务刷屏主对话、之后又用不到） | Route it through a **subagent** |
| You want something to happen every time without asking（你想让某件事每次都发生、不用开口） | Write a **hook** |
| A second repository needs the same setup（第二个仓库也需要同一套配置） | Package it as a **plugin** |

值得注意的**顺序**：先 CLAUDE.md（约定）→ 再 skill（重复流程，占了两个触发器）→ 然后 MCP（外部连接）→ 再代码智能插件 / subagent（性能与隔离）→ hook（确定性）→ 最后 plugin（跨仓库复用）。这张表把"什么时候用哪种"从抽象原则变成了可对号入座的信号；官方还补了一句——"The same triggers tell you when to update what you already have"（同样的触发信号也告诉你什么时候该升级已有的配置）。

### 8.3 三个最容易踩的坑

- **别用 MCP 去解决"知识"问题。** MCP 管"连接与数据访问"，skill 管"怎么用好"。想给 Claude 塞领域知识，答案是 skill，不是再挂一个 MCP server。
- **别用 skill 去"保证"某事不发生。** 指令只是请求，不是保证；官方博客原话："A real guardrail needs to be deterministic, and the enforcement methods are hooks and permissions."（真正的护栏必须是确定性的，执行手段是 hooks 和 permissions。）要强制，用 hook / permissions。
- **别一上来就写插件。** 官方明说先在 `.claude/` 裸配快速迭代，等要分享/跨仓库了再转插件（"Start with standalone configuration in `.claude/` for quick iteration, then convert to a plugin when you're ready to share"）。插件多了命名空间（`/my-plugin:hello`）和版本管理，单独开发时是纯负担。

最后收尾一句官方原话（`features-overview` 的 "Combine features" 一节），把整个扩展层的分工关系讲清楚了：

> "Each extension solves a different problem: CLAUDE.md handles always-on context, skills handle on-demand knowledge and workflows, MCP handles external connections, subagents handle isolation, and hooks handle automation. Real setups combine them based on your workflow."
> （每种扩展解决不同的问题：CLAUDE.md 管常驻上下文，skills 管按需知识与工作流，MCP 管外部连接，subagents 管隔离，hooks 管自动化。**真实配置是按你的工作流组合它们。**）

---

## 参考来源

本文内容综合以下资料整理，所有引用均于 2026-08-04 通过 web_fetch 抓取官方 Markdown 原文核对（`code.claude.com/docs/en/*.md` 与 `claude.com/blog/...`）：

- **Extend Claude Code（features-overview）** — https://code.claude.com/docs/en/features-overview
  （核心对比页：扩展层总览四句定位、"Match features to your goal" 总表、"[Plugins] are the packaging layer"、MCP vs Skill / Hook vs Skill 对比 tab、触发信号表、上下文成本表、组合套路表、"Each extension solves a different problem"）
- **Connect Claude Code to tools via MCP** — https://code.claude.com/docs/en/mcp
  （MCP 定位、"copying data into chat from another tool" 触发信号、六类典型用途、tool search 与自动重连）
- **Automate actions with hooks** — https://code.claude.com/docs/en/hooks-guide
  （hooks 定义与"deterministic control"、enforce rules / automate / integrate 三大用途、prompt/agent-based hooks）
- **Hooks reference** — https://code.claude.com/docs/en/hooks
  （hook 类型含 `mcp_tool`、PreToolUse exit code 2 即阻塞工具调用、JSON 输出 `permissionDecision: "deny"`、五类 hook 的输入输出约定）
- **Configure permissions** — https://code.claude.com/docs/en/permissions
  （"PreToolUse hooks run before the permission prompt"、"Hook decisions don't bypass permission rules"——护栏的边界与 hooks/permissions 分工）
- **Extend Claude with skills** — https://code.claude.com/docs/en/skills
  （"Create a skill when you keep pasting the same instructions…" 判据、"the most flexible extension"、skill 正文按需加载、reference vs action、自定义命令已并入 skills、`disable-model-invocation`）
- **Create plugins** — https://code.claude.com/docs/en/plugins
  （插件定位与目录结构、Standalone vs Plugin 对照表、"Use plugins when" 清单、"先独立配置再转插件" 路径、命名空间机制）
- **Claude 官方博客：Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents** — https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more
  （"CLAUDE.md is for facts…" / "Procedures belong in skills"、"Use hooks for anything that should happen deterministically"、"an instruction is the wrong tool"、"A real guardrail needs to be deterministic, and the enforcement methods are hooks and permissions"、"The model choosing to run a formatter is different from the formatter running automatically"、"Skills and hooks are also the building blocks of designing agent loops"）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 与上下文隔离）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（MCP 接入既有搜索/索引的姿势、LSP 代码智能插件）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（CLAUDE.md 与上下文管理）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
