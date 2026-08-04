# Claude Code 官方建议什么时候用 MCP、hooks、skills、plugins？

> **一句话总结**：官方文档把四种扩展方式定位成"各解决一类不同问题"的四层工具，判据非常具体——**skills** 用于"可复用知识 + 可调用的工作流"（你反复把同一段流程贴进对话、或 CLAUDE.md 里某一段从"事实"长成了"步骤"时）；**MCP** 用于"连外部服务"（你老是从某个工具/浏览器把数据复制进对话时）；**hooks** 用于"每次都必须以相同方式发生的确定性自动化与护栏"（"每次 X 都要做 Y""绝不能做 Z"这类规则，官方点名要用 hook 而不是提示词）；**plugins 不是第四种功能扩展，而是"打包层"**——它把 skills/agents/hooks/MCP servers 打包成一个可安装单元，官方原话："Use plugins when you want to reuse the same setup across multiple repositories or distribute to others via a marketplace"（当你想跨仓库复用同一套配置、或通过 marketplace 分发给别人时用插件）。真实的工程配置不是四选一，而是**组合**使用。
>
> 本文基于 Claude Code 官方文档 `Extend Claude Code`、`Extend Claude with skills`、`Connect Claude Code to tools via MCP`、`Automate actions with hooks`、`Create plugins`、`Discover and install prebuilt plugins`，以及官方博客 `Steering Claude Code` 整理，文末附参考来源。

## 一、先搞懂：plugins 不是"第四种功能"，而是前三者的"打包层"

官方把所有扩展统称为"扩展层"（extension layer），在 `Extend Claude Code` 这一页 overview 里，每种扩展各占一句话：

> "Skills add reusable knowledge and invocable workflows."
> （Skills：增加**可复用的知识**与**可调用的工作流**。）
>
> "MCP connects Claude to external services and tools."
> （MCP：把 Claude **连到外部服务和工具**。）
>
> "Hooks run your script, HTTP request, prompt, or subagent when Claude Code reaches a lifecycle event."
> （Hooks：当 Claude Code 到达某个**生命周期事件**时，运行你的脚本、HTTP 请求、提示词或 subagent。）
>
> "Plugins and marketplaces package and distribute these features."
> （Plugins 与 marketplaces：**打包并分发**上面这些功能。）

注意最后一句的关键词：**package and distribute（打包并分发）**。在官方的坐标系里，plugins 和 skills/MCP/hooks 不在同一个维度上——前三种是"功能扩展"，plugin 是"把功能打包成可分发的单元"。`Create plugins` 文档开篇也直接点破：

> "Plugins let you extend Claude Code with custom functionality that can be shared across projects and teams."
> （Plugins 让你用可以在项目与团队之间**共享**的自定义功能来扩展 Claude Code。）

这就是"什么时候用 plugins"的总答案：**当你要复用 / 分享 / 分发时**。`Extend Claude Code` 里那句说得最直白：

> "Plugins are the packaging layer. A plugin bundles skills, hooks, subagents, and MCP servers into a single installable unit. … Use plugins when you want to reuse the same setup across multiple repositories or distribute to others via a marketplace."
> （Plugins 是**打包层**。一个 plugin 把 skills、hooks、subagents 和 MCP servers 打包成**一个可安装单元**。……当你想要**跨多个仓库复用同一套配置**、或通过 marketplace **分发给别人**时，用 plugins。）

这一页还给了每种扩展"定位 + 何时用 + 例子"的对照表（节选四行）：

| 扩展 | 它做什么 | 什么时候用（官方原话） | 官方例子 |
|---|---|---|---|
| **Skill** | Instructions, knowledge, and workflows Claude can use | Reusable content, reference docs, repeatable tasks（可复用内容、参考文档、可重复任务） | `/deploy` runs your deployment checklist |
| **MCP** | Connect to external services | External data or actions（外部数据或动作） | Query your database, post to Slack, control a browser |
| **Hook** | Script, HTTP request, prompt, or subagent triggered by events | Automation that must run on every matching event（必须在每个匹配事件上都跑的自动化） | Run ESLint after every file edit |
| **Plugins** | （打包层）Package and distribute the features above | Reuse the same setup across repositories / distribute via a marketplace | 把 skills、hooks、MCP 打包成一个 `/my-plugin:xxx` 单元 |

## 二、Skills：什么时候用

`Extend Claude with skills` 文档开头就给了"该不该建 skill"的判据，而且是两个很具体的信号：

> "Create a skill when you keep pasting the same instructions, checklist, or multi-step procedure into chat, or when a section of CLAUDE.md has grown into a procedure rather than a fact."
> （当你**反复把同一段指令、清单、或多步流程贴进对话**，或当 CLAUDE.md 的某一段**从"事实"长成了"步骤"**时，就创建一个 skill。）

它紧接着补了选 skill 而不选 CLAUDE.md 的成本理由：

> "Unlike CLAUDE.md content, a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."
> （和 CLAUDE.md 的内容不同，skill 的**正文只在被用到时才加载**，所以很长的参考资料在你需要之前几乎不花成本。）

官方把 skill 分成两种形态，这也直接决定了"什么时候用"：

| 形态 | 是什么 | 典型例子 |
|---|---|---|
| **Reference（参考型）** | 提供 Claude 全程使用的知识 | API style guide（API 风格指南）、数据模型说明 |
| **Action（动作型）** | 告诉 Claude 去做一件具体的事，通常用 `/<name>` 主动调用 | `/deploy` 跑你的部署检查单、`/release` 发布流程 |

对应到"该由谁触发"，官方推荐对**有副作用**的流程型 skill 加 `disable-model-invocation: true`，让它只在你自己调用时才跑、平时不占上下文。

## 三、MCP：什么时候用

`Connect Claude Code to tools via MCP` 开头的判据非常"症状化"：

> "Connect a server when you find yourself copying data into chat from another tool, like an issue tracker or a monitoring dashboard."
> （当你**发现自己正从另一个工具往对话里复制数据**——比如从 issue 跟踪器或监控面板——就该连一个 MCP server。）

> "Once connected, Claude can read and act on that system directly instead of working from what you paste."
> （连上之后，Claude 就能**直接读、直接操作**那个系统，而不是从你粘贴的内容里干活。）

换言之，**"从某个工具复制数据进对话"这个动作，就是官方给的 MCP 触发器**。复制意味着两件事：一、那个系统 Claude 看不到；二、粘贴的过程既慢又丢信息。MCP 把"粘贴"升级成"直接读 + 直接操作"。

官方给的典型用途包括：从 issue tracker 实现功能、查数据库、看 Sentry/Statsig 监控数据、按 Figma 设计稿改邮件模板、用 Gmail 草稿做自动化工作流等——共同点是**外部系统 + 数据/动作**。

MCP 与 skill 的分工容易混淆（连同一个外部系统时，到底用谁？），官方专门开了一个 "MCP vs Skill" 对比 tab，核心是两句话：

> "MCP connects Claude to external services. Skills extend what Claude knows, including how to use those services effectively."
> （MCP 连接外部服务；Skills 扩展 Claude 的**知识**，包括怎么**用好**那些服务。）

> "MCP gives Claude purpose-built tools for an external system, with the connection and authentication handled by the server."
> （MCP 给 Claude 提供面向某个外部系统的**专用工具**，连接与认证由 server 处理。）
>
> "Skills give Claude knowledge about how to use those tools effectively, plus workflows you can trigger with `/<name>`."
> （Skills 给 Claude 提供"怎么用好那些工具"的知识，外加你能用 `/<name>` 触发的工作流。）

官方的例子是黄金组合：**MCP server 连上你的数据库，skill 教 Claude 你的数据模型、常用查询模式、不同任务该用哪些表**。注意，两者不是"二选一"，而是"连接 vs 用法"两层。

## 四、Hooks：什么时候用

`Automate actions with hooks` 开篇点出了 hook 与 skill 最根本的区别——**确定性（determinism）**：

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you deterministic control: certain actions always happen rather than relying on the LLM to choose to run them. Use hooks to enforce project rules, automate repetitive tasks, and integrate Claude Code with your existing tools."
> （Hooks 是用户自定义的 shell 命令。Claude Code 在生命周期里的特定节点运行它们，这给了你**确定性控制**：某些动作**一定会发生**，而不是靠 LLM 自己决定去跑。用 hooks 来**强制执行项目规则**、自动化重复任务、并和你的既有工具集成。）

注意这句里最关键的部分：**rather than relying on the LLM to choose to run them**（而不是依赖 LLM 决定要不要跑）。skill 和 CLAUDE.md 里的指令对 Claude 来说都是"请求"（request），可能被遵守也可能不被遵守；hook 则是"强制执行"。

官方 `Extend Claude Code` 的 "Hook vs Skill" 对比 tab 给了最干脆的分界线：

> "Use a hook when the action must happen the same way every time and doesn't need Claude to think. For example: format on save, reject `rm -rf /`, post a Slack message when a session ends."
> （当某个动作**每次都必须以相同方式发生、且不需要 Claude 思考**时，用 hook。例如：保存时自动格式化、拒绝 `rm -rf /`、会话结束时发一条 Slack 消息。）

> "Use a skill when Claude should decide how to apply the steps, or when the content is knowledge rather than a script. For example: a `/release` checklist, your API style guide, a debugging playbook."
> （当**应该由 Claude 决定怎么执行**、或内容是**知识而不是脚本**时，用 skill。例如：`/release` 检查单、API 风格指南、调试手册。）

以及一条经常被忽略的"护栏"建议——**凡是"必须每次生效"的规则，官方点名要用 hook，而不是写进提示词**：

> "Put guardrails in hooks. An instruction like 'never edit `.env`' in CLAUDE.md or a skill is a request, not a guarantee. A `PreToolUse` hook that blocks the edit is enforcement. If a rule must hold every time, make it a hook rather than a prompt instruction."
> （把护栏放进 hooks。写在 CLAUDE.md 或 skill 里的"绝不编辑 `.env`"只是一条**请求**，不是**保证**。一个 `PreToolUse` hook 直接拦下那次编辑，才是**强制执行**。如果一条规则必须每次都生效，就把它做成 hook，而不是一条提示词指令。）

另一个差异是上下文成本：hooks 默认**零成本**——官方原文 "Nothing (runs externally)"、"Zero, unless hook returns additional context"（除非 hook 返回额外内容，否则不占上下文），这让它特别适合 lint、日志、通知这类"副作用"。

官方还说明，如果某些决策需要"判断"而不是"确定规则"，hook 也有带模型的变体：

> "For decisions that require judgment rather than deterministic rules, you can also use prompt-based hooks or agent-based hooks that use a Claude model to evaluate conditions."
> （对需要**判断**而非确定性规则的决定，你也可以用 prompt-based 或 agent-based hooks，让一个 Claude 模型来评估条件。）

## 五、Plugins：什么时候用

"什么时候用 plugin"在 `Create plugins` 文档里被做成了一张**直接对比表**，对手是"独立配置"（standalone configuration，即 `.claude/` 目录）：

| 方式 | skill 命名 | 官方：最适合什么 |
|---|---|---|
| **Standalone**（`.claude/` 目录） | `/hello` | Personal workflows, project-specific customizations, quick experiments（个人工作流、项目级定制、快速实验） |
| **Plugins**（自带 skills/agents/hooks/MCP 的独立目录，含 `.claude-plugin/plugin.json` 清单） | `/plugin-name:hello` | Sharing with teammates, distributing to community, versioned releases, reusable across projects（分享给队友、分发给社区、带版本发布、跨项目复用） |

随后两条判断清单：

> "Use standalone configuration when: You're customizing Claude Code for a single project / The configuration is personal and doesn't need to be shared / You're experimenting with skills or hooks before packaging them / You want short skill names like `/hello` or `/deploy`."
> （**用独立配置**，当：你只是给单个项目定制、配置是私人的不需要分享、你在打包之前**实验** skills/hooks、你想要 `/hello` 这种短 skill 名。）

> "Use plugins when: You want to share functionality with your team or community / You need the same skills/agents across multiple projects / You want version control and easy updates for your extensions / You're distributing through a marketplace / You're okay with namespaced skills like `/my-plugin:hello`."
> （**用 plugins**，当：你要把功能分享给团队或社区、你需要在**多个项目**里用同一套 skills/agents、你想要**版本控制**和方便更新、你要通过 marketplace 分发、你能接受 `/my-plugin:hello` 这种带命名空间的 skill 名。）

官方还给了一条非常实用的迭代路径建议：

> "Start with standalone configuration in `.claude/` for quick iteration, then convert to a plugin when you're ready to share."
> （先在 `.claude/` 里用独立配置快速迭代，**准备好分享时再转成 plugin**。）

命名空间（namespacing）是理解 plugin 的关键机制：plugin 里的 skill 一律带 `plugin-name:` 前缀（如 `/my-plugin:hello`），目的是**防止多个插件之间 skill 重名冲突**。这也是用 plugin 的隐性代价——skill 名变长。

## 六、官方给的"触发词"速查表：按症状找该加什么

`Extend Claude Code` 里有一张可能是全文最实用的表——"Build your setup over time"（逐步搭建你的配置）。它不按功能讲，而按**症状**讲：你遇到哪个现象，就加哪个扩展。完整八行：

| 触发信号（官方原话） | 该加什么 |
|---|---|
| Claude gets a convention or command wrong twice（约定/命令错了两遍） | Add it to CLAUDE.md |
| You keep typing the same prompt to start a task（每次开工都要敲同一段提示词） | Save it as a user-invocable **skill** |
| You paste the same playbook or multi-step procedure into chat for the third time（同一套流程第三次贴进对话） | Capture it as a **skill** |
| You keep copying data from a browser tab Claude can't see（老是从 Claude 看不到的浏览器标签页复制数据） | Connect that system as an **MCP server** |
| Claude reads many files to find where a symbol is defined or used（Claude 为找一个符号读了一堆文件） | Install a code intelligence **plugin**（LSP） |
| A side task floods your conversation with output you won't reference again（边角任务把大段不会再引用的输出灌进对话） | Route it through a **subagent** |
| You want something to happen every time without asking（想让某件事每次自动发生、不用开口） | Write a **hook** |
| A second repository needs the same setup（第二个仓库也需要同一套配置） | Package it as a **plugin** |

注意这八行里：**skills 占了两个触发器（重复的开工提示词 + 反复粘贴的流程），plugin 也有两个（代码智能、跨仓库复用），MCP、hook、CLAUDE.md、subagent 各一个**。这本身就在暗示——skills 是官方认为**最常该加**的扩展。

## 七、容易混淆的两对边界，官方是这么分的

### MCP vs Skill

一句话：**MCP 管"连接"，skill 管"用法"。** MCP 提供连到外部系统的专用工具（连接和认证都在 server 端处理）；skill 提供"怎么用好那些工具"的知识，外加可用 `/<name>` 触发的工作流。二者经常成对出现，不是竞争关系。官方组合示例：

| 组合 | 怎么工作 | 例子 |
|---|---|---|
| **Skill + MCP** | MCP 提供连接；skill 教 Claude 怎么用好它 | MCP 连数据库，skill 记录 schema 与查询模式 |

### Hook vs Skill

| 维度 | Hook | Skill |
|---|---|---|
| 运行方式 | 跑 shell 命令 / HTTP 请求 / LLM prompt / subagent | Claude **读指令并按自己的理解执行** |
| 触发 | 生命周期事件（`PostToolUse`、`SessionStart` 等），事件发生必触发 | 你打 `/<name>`，或 Claude 根据 description 判断是否相关 |
| 确定性 | 必然发生（官方原话 "the trigger is guaranteed"） | 由 Claude 解释，结果可能不同 |
| 上下文成本 | 零（除非 hook 返回输出） | description 每会话加载；正文用时加载 |
| 官方 Best for | Linting after edits, blocking unsafe commands, logging, notifications | Workflows that need reasoning, reference material, multi-step tasks |

判据只有一句：**动作每次都要一模一样、不需要 Claude 思考 → hook；需要 Claude 决定怎么执行、或内容是知识 → skill。** 护栏（guardrail）默认放 hook，因为提示词只是"请求"，hook 才是"执行"。

## 八、落地：一张总表 + 一个组合示例

把官方几篇文档收拢成一张"什么时候用什么"的最终对照表：

| 扩展方式 | 官方定位 | 官方判据（一句话） | 官方例子 |
|---|---|---|---|
| **Skills** | 可复用知识 + 可调用工作流（"最灵活的扩展"） | 反复粘贴同一段流程；CLAUDE.md 某段长成了步骤 | `/deploy`、API 风格指南、调试手册 |
| **MCP** | 连外部服务与工具 | 老是从某个工具往对话复制数据 | 查数据库、发 Slack、控制浏览器 |
| **Hooks** | 生命周期事件上的确定性自动化 | 每次都必须以相同方式发生、不需要 Claude 思考 | 保存即格式化、拦 `rm -rf /`、会话结束发通知 |
| **Plugins** | 打包层：把 skills/agents/hooks/MCP 打包分发 | 要跨仓库复用同一套配置、或分享/分发 | `/my-plugin:hello`、LSP 代码智能插件 |

官方对"要不要全都配上"的回答是：**不需要一次配齐，按症状逐步加**，而且最终是**组合**关系：

> "Each extension solves a different problem: CLAUDE.md handles always-on context, skills handle on-demand knowledge and workflows, MCP handles external connections, subagents handle isolation, and hooks handle automation. Real setups combine them based on your workflow."
> （每种扩展解决不同的问题：CLAUDE.md 管常驻上下文，skills 管按需知识与工作流，MCP 管外部连接，subagents 管隔离，hooks 管自动化。**真实配置是按你的工作流组合它们。**）

官方博客 `Steering Claude Code` 给出的经验法则与文档一致（本文以转述引用）：**"每次 X 都做 Y""绝不 Z"这类话如果出现在 CLAUDE.md 里，那它就该是个 hook**——真正的护栏需要确定性，执行手段是 hooks + permissions；而 procedural（流程式）内容——部署流程、发布检查单、审查流程——应该放进 skill 而不是 CLAUDE.md。

最后一个务实建议，直接照抄官方：**先在 `.claude/` 里用独立配置（skill/hook）快速迭代，等"第二个仓库也需要同一套"时再打包成 plugin。** 大多数个人项目根本走不到 plugin 那一步。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 抓取官方 Markdown 原文 `code.claude.com/docs/en/*.md` 核对获取）：

- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （本文主干：扩展层定位、"Match features to your goal" 对照表、"Build your setup over time" 触发器速查表、"MCP vs Skill / Hook vs Skill" 对比 tab、"Plugins are the packaging layer"、上下文成本表、组合模式）
- **Extend Claude with skills** — https://code.claude.com/docs/en/skills
  （"Create a skill when..." 判据、skill 正文按需加载的成本机制、reference vs action 两种形态）
- **Connect Claude Code to tools via MCP** — https://code.claude.com/docs/en/mcp
  （"Connect a server when you find yourself copying data into chat" 判据、MCP 典型用途）
- **Automate actions with hooks** — https://code.claude.com/docs/en/hooks-guide
  （"deterministic control" 定位、hooks 的用途、prompt/agent-based hooks、HTTP hooks）
- **Create plugins** — https://code.claude.com/docs/en/plugins
  （"plugins vs standalone configuration" 对照表、两条判断清单、"start standalone, convert to plugin" 建议、命名空间机制）
- **Discover and install prebuilt plugins** — https://code.claude.com/docs/en/discover-plugins
  （plugin 能打包哪些组件、官方 marketplace 的分类：代码智能 LSP / 外部集成（捆绑 MCP servers）/ 安全审查 / 开发工作流）
- **官方博客：Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents** — https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more
  （"真正护栏需要确定性，执行手段是 hooks 和 permissions"、procedural 内容放 skill 的经验法则；本文以转述引用，未逐字引用）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 的官方判据与"主对话 vs subagent"对照，扩展层第 5 块）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（MCP 作为接入既有代码搜索索引的姿势、LSP 代码智能插件）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
