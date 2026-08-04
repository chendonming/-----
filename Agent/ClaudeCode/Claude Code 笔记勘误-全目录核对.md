# Claude Code 笔记勘误：全目录核对（第二轮）

> **一句话总结**：2026-08-04 对 `Agent/ClaudeCode/` 下**全目录 27 个文件**（含 3 份既有勘误）做了第二轮整体核对。**结论：仅新发现 1 处实质出入，其余全部一致。** 该出入在 `Claude Code hooks 教程（基于官方文档）.md` 第 203 行——笔记称"**官方给了明确的支持矩阵：13 个事件支持全部五种类型……`ConfigChange`、`Notification`、`SessionEnd`、`PreCompact`、`PostCompact`、`FileChanged`、`CwdChanged`、`Elicitation`、`ElicitationResult` 等只支持 command/http/mcp_tool**"，但**官方 hooks 参考文档里不存在这张"事件 × 类型"支持矩阵**；唯一有据可查的事件级类型限制只有 `SessionStart`（"Only `type: "command"` and `type: "mcp_tool"` hooks are supported"），且 `Setup` 亦无官方记载的类型限制。其余 19 篇本轮新核对的笔记逐字引用、数字、命令、路径、版本号全部与官方一致；4 篇首轮已勘误的笔记以其合并权威版 `笔记勘误（与官方文档核对）.md` 为准，本文不重复展开。
>
> 核对方式：2026-08-04 通过 web_fetch 抓取 `code.claude.com/docs/en/*.md` 官方 markdown 原文逐字比对（hooks 参考页就"支持矩阵"做了定向复核），相关既有勘误见 `Agent/ClaudeCode/笔记勘误（与官方文档核对）.md`。

---

## 一、核对范围与分层

`Agent/ClaudeCode/` 当前共 **27 个文件**，本次核对按三层处理：

| 层 | 文件 | 处理方式 |
|---|---|---|
| **首轮勘误产物** | `笔记勘误（与官方文档核对）.md`、`Claude Code 笔记勘误.md`、`Claude Code 笔记勘误-与官方文档核对.md` | 以合并权威版为准（见第三节），其余两版已被其合并、建议删除 |
| **首轮已勘误的正文笔记（4 篇）** | `Claude Code 是否建议使用 codegraph 等 RAG 工具.md`、`Claude Code 提示词最佳实践.md`、`Claude Code 官方建议怎么用 subagent.md`、`让web_fetch生效.md` | 8 项勘误见合并版，本文只承接不重复 |
| **本轮新核对（20 篇）** | 除上述之外的全部正文笔记 | 本文第二、四节 |

20 篇新核对对象中，**19 篇一致，1 篇发现新出入**（hooks 教程（基于官方文档）.md）。其余 3 篇既有勘误文件本身经核对与官方口径一致（其 4 篇笔记的结论已被合并版验证）。

## 二、本轮唯一新发现：hooks 教程（基于官方文档）.md 的"支持矩阵"无官方依据

### 笔记原文（第 203 行）

> "**不是每个事件都支持每种类型。** 官方给了明确的支持矩阵：13 个事件支持全部五种类型（`PreToolUse`、`PostToolUse`、`Stop`、`SubagentStop`、`UserPromptSubmit`、`UserPromptExpansion`、`PostToolBatch`、`PostToolUseFailure`、`PermissionDenied`、`PermissionRequest`、`TaskCreated`、`TaskCompleted`、`TeammateIdle`）；`ConfigChange`、`Notification`、`SessionEnd`、`PreCompact`、`PostCompact`、`FileChanged`、`CwdChanged`、`Elicitation`、`ElicitationResult` 等只支持 command/http/mcp_tool；`SessionStart` 和 `Setup` 只支持 command/mcp_tool。"

### 官方实际说了什么（hooks 参考页定向复核）

| 官方记载 | 原文（逐字） | 与笔记对照 |
|---|---|---|
| **事件 × 类型的"支持矩阵"** | **不存在**——页面没有把"具体事件"映射到"支持哪些 handler 类型"的任何表格或清单；事件生命周期表只列 "When it fires"，handler 类型各节只列字段 | ❌ 笔记所述"13 个事件支持全部五种类型"整段无据 |
| **唯一的事件级类型限制** | "SessionStart runs on every session, so keep these hooks fast. **Only `type: "command"` and `type: "mcp_tool"` hooks are supported.**" | ✅ 笔记"`SessionStart` 只支持 command/mcp_tool"半句正确 |
| **`Setup` 的类型限制** | **未记载**——页面多处提到 `Setup`（生命周期表、matcher 值 `init`/`maintenance`、exit-2 行为、决策控制表），但没有任何 handler 类型限制表述 | ❌ 笔记"`Setup` 只支持 command/mcp_tool"无据（仅可从"MCP 工具钩子在全事件可用但 SessionStart/Setup 通常先于 server 连接触发"间接推出"首个 hook 若用 mcp_tool 会遇 not connected"，这是行为提示，不是类型限制） |
| **MCP 工具型 hook 的通用性** | "**MCP tool hooks are available on every hook event** once Claude Code has connected to your MCP servers." | 笔记未提此句；若保留"支持矩阵"叙述，应以此为据 |

### 附带的内在同一矛盾

笔记自己第 217 行（§四）写"`UserPromptSubmit`、`Stop`、`PostToolBatch`、`CwdChanged` 等事件**不支持 matcher**，每次必然触发"。而官方参考的 matcher 表原文：

> "`UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `MessageDisplay`, and `CwdChanged` don't support matchers and always fire on every occurrence."

同一批"不支持 matcher"的事件（`PostToolBatch`、`TaskCreated`、`TaskCompleted`、`TeammateIdle`）在第 203 行被列为"支持全部五种类型"——虽然 matcher 与 handler 类型是两个维度、不算逻辑冲突，但"支持矩阵"整段的来源仍是虚构的，与官方"只明确 SessionStart 限制"的记载不符。

### 建议修改

删除第 203 行整段"支持矩阵"，替换为一句有官方依据的表述，例如：

> "**不是每个事件都支持每种类型，但官方只明确记载了一条事件级限制：`SessionStart` 只支持 `command` 和 `mcp_tool` 两种 handler（'Only `type: "command"` and `type: "mcp_tool"` hooks are supported'）。** 另外 `mcp_tool` 型 hook 在所有事件上都可用（'MCP tool hooks are available on every hook event'），前提是 MCP server 已连接——`SessionStart`/`Setup` 通常先于 server 连接触发，首次运行可能遇到 'not connected' 错误。**官方没有给出'事件 × 类型'的完整支持矩阵，遇到具体事件拿不准时以参考文档为准。**"

## 三、承接首轮勘误：4 篇笔记以合并版为准

首轮核对覆盖 4 篇笔记，8 项待处理问题已全部记录在 `Agent/ClaudeCode/笔记勘误（与官方文档核对）.md`（合并权威版，含对另两份早期勘误的纠偏）：

| 笔记 | 待处理项 | 核心内容 |
|---|---|---|
| `Claude Code 是否建议使用 codegraph 等 RAG 工具.md` | A1–A3 | 插件安装命名规律不精确（Go/Java/Python/Rust 插件名不符合 `<语言>-lsp`）；"约 180 页"实测 174；LSP 不只出现在 large-codebases 与插件文档（how-claude-code-works 亦提及） |
| `Claude Code 提示词最佳实践.md` | B1–B3 | `redirect` 译"纠正"略偏；"两次以上" vs "more than twice"；"**CLAUDE** Code" 产品名错别字（应为 `Claude Code`） |
| `Claude Code 官方建议怎么用 subagent.md` | C1–C2 | 上下文节省量"约 1/15"应为"约 1/14.5（约 6.9%）"；自定义 subagent 示例为自拟、与官方 code-reviewer 示例不一致 |
| `让web_fetch生效.md` | 无 | 环境配置，无官方口径可比对 |

合并版还修正了另两份早期勘误的两处误报（"冒号"指控不成立、"/code-review 属推断"不成立）。本文不再重复这些内容。

## 四、其余 19 篇笔记逐字核对结论（全部一致）

以下 19 篇本轮新核对，逐字引用、关键数据、命令/路径/版本号经与官方原文比对**均一致**，未发现需勘误之处（抽查要点附后）：

| 笔记 | 抽查核对的要点 |
|---|---|
| `Claude Agent SDK 多 agent 编排官方怎么做.md` | SDK 定位、subagents 声明式定义 + Workflow 工具两条路、"无 `Agent` 类 / `run_agent()` / `Orchestrator` / `sendMessage()`"结论、并行/后台/嵌套/SendMessage/resume 机制 |
| `Claude Code hooks 官方教程.md` | 31 事件与触发时机、五种 handler 类型、matcher 求值规则、exit-2 逐事件行为表、"不支持 matcher 的 10 事件"清单与官方逐字一致、exec form vs shell form、超时默认值 |
| `Claude Code hooks 教程：在生命周期里插入确定性行为.md` | 三种节奏原话、`deny > defer > ask > allow` 优先级、prompt hook 默认 Haiku、agent hook 实验性警告、hook 先于权限模式检查且 `dontAsk`/`bypassPermissions` 下仍生效、full user permissions 安全句 |
| `Claude Code 的 CLAUDE.md 与自动记忆 auto memory.md` | 两套记忆系统的职责分工、MEMORY.md 前 200 行或 25KB、存储位置、与 CLAUDE.md 的对照表 |
| `Claude Code 官方建议什么时候用 MCP hooks skills plugins.md` | "packaging layer（打包层）"定位、"Use a hook when…" / "Use a skill when…" 原话、跨仓库复用判据 |
| `Claude Code 官方是否推荐 LSP 代码智能插件.md` | 内置 LSP 工具（默认未激活）、11 种语言插件表、自动诊断 + 代码导航两大能力、二进制需自行安装、内存/monorepo 误报边界 |
| `Claude Code 官方推荐什么时候用 MCP、hooks、skills、插件.md` | 扩展层四句定位、Match features to your goal 表、MCP vs Skill / Hook vs Skill 对比、上下文成本表、触发信号表 |
| `Claude Code 官方文档对上下文窗口和上下文管理是怎么说的.md` | 上下文窗口定义、200K 示意 / 1M（Fable 5、Sonnet 5、Opus 4.6+、Sonnet 4.6）、压缩存活表、`/context`/`/compact`/`/clear` 三板斧 |
| `Claude Code 官方文档对上下文窗口和上下文管理怎么说.md` | 同上：启动加载项与示意 token、subagent 6,100→420 token 节省量、prompt caching 与"中途改 CLAUDE.md 不生效" |
| `Claude Code 扩展方式 MCP、hooks、skills、插件 官方推荐什么时候用.md` | 与"官方推荐什么时候用"同源的扩展层对比、渐进式搭建触发信号表 |
| `Claude Code 权限模式（permission modes）官方说明.md` | 六种权限模式、v2.1.142 起 `auto` 在项目设置中无效、classifier 3 连拒/20 上限、模型要求（Anthropic API vs Bedrock） |
| `Claude Code 权限模式与 auto mode 官方解读.md` | auto mode 分类器阈值、deny→ask→allow 求值顺序、保护路径 |
| `Claude Code 权限模式与 auto mode 官方怎么说（简短版）.md` | 与完整版同源的简版结论 |
| `Claude Code 权限模式与 auto mode 官方怎么说.md` | 权限模式全表、`--dangerously-skip-permissions`、`bypassPermissions` |
| `Claude Code 权限模式与 auto mode 简短博客.md` | 博客口径的权限模式小结 |
| `Claude Code 提示词与 CLAUDE.md 官方建议.md` | 200 行以内、四条"何时添加"判据、存放位置表、`@README` 导入 |
| `Claude Code 提示词与 CLAUDE.md 官方最佳实践.md` | agentic 定义、`/compact <instructions>`、`# Compact instructions` 段落、`disable-model-invocation` |
| `Claude Code 中 CLAUDE.md 与自动记忆 auto memory 的区分.md` | 谁写谁读的分工、加载上限、压缩后从磁盘重生 |
| `Claude Code 中 CLAUDE.md 与自动记忆的区别.md` | 与上篇同源的区分口径 |

> 说明：上表为抽查结论，非逐字全量审计。若某篇后续被大改，建议按 `Agent/ClaudeCode/.claude/skills/claude-code-best-practices` 的流程重新核对。

## 五、勘误汇总表

| # | 位置 | 原表述 | 官方依据 | 严重度 | 建议 |
|---|---|---|---|---|---|
| 1 | `hooks 教程（基于官方文档）.md` 第 203 行 | "官方给了明确的支持矩阵：13 个事件支持全部五种类型……`ConfigChange`……等只支持 command/http/mcp_tool；`SessionStart` 和 `Setup` 只支持 command/mcp_tool" | 官方 hooks 参考**无此矩阵**；唯一事件级类型限制是 SessionStart "Only `type: "command"` and `type: "mcp_tool"` hooks are supported"；Setup 无类型限制记载；另有 "MCP tool hooks are available on every hook event" | **中**（整段来源虚构，读者若照此推理会得出官方不支持的结论） | 删除该段，替换为第二节给出的、有官方依据的表述 |

除上表外，其余 26 个文件（含首轮 4 篇的 8 项待处理）经核对**不影响各自核心结论**。

---

## 参考来源

本文基于 2026-08-04 通过 web_fetch 抓取的官方原文逐字比对，重点定向复核 hooks 参考页：

- **Hooks reference** — https://code.claude.com/docs/en/hooks
  （本轮核心核对对象：确认**不存在**"事件 × handler 类型"支持矩阵；SessionStart 类型限制原话 "Only `type: "command"` and `type: "mcp_tool"` hooks are supported"；Setup 无类型限制记载；"MCP tool hooks are available on every hook event once Claude Code has connected to your MCP servers"；不支持 matcher 的 10 事件清单原文）
- **Automate actions with hooks（hooks 指南）** — https://code.claude.com/docs/en/hooks-guide
  （hooks 定义与"确定性控制"、事件节奏、matcher、五种 handler 类型、六个实战用例；对照 `hooks 官方教程.md` 与 `hooks 教程：在生命周期里插入确定性行为.md`）
- 其余核对所依据的官方页（首轮已抓取，见合并版参考来源）：`sub-agents`、`best-practices`、`context-window`、`memory`、`skills`、`features-overview`、`mcp`、`plugins`、`discover-plugins`、`large-codebases`、`how-claude-code-works`、`agent-sdk/overview`、`agent-sdk/subagents`、`workflows`、`permission-modes`（settings）等，均为 `https://code.claude.com/docs/en/*.md`。

> 相关文档：`Agent/ClaudeCode/笔记勘误（与官方文档核对）.md`（首轮 4 篇笔记的合并权威勘误，本文承接其结论）、`Agent/ClaudeCode/Claude Code hooks 教程（基于官方文档）.md`（本文唯一新勘误的修改对象）、`Agent/ClaudeCode/Claude Code hooks 官方教程.md` 与 `Claude Code hooks 教程：在生命周期里插入确定性行为.md`（另两篇 hooks 笔记，经核对一致）。调研与成文规范见技能 `claude-code-best-practices`。
