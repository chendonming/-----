# Claude Code 笔记勘误：四篇笔记与官方文档逐字核对

> **一句话总结**：2026-08-04 将 `Agent/ClaudeCode/` 下四篇笔记的全部英文引用、关键数据与功能描述，与官方文档（`code.claude.com/docs/en/*.md` 原始 markdown）逐字比对。**结论：整体高度准确，四篇笔记里"官方原话"级别的引用几乎全部核对无误**；仅发现 3 处次要出入（一处近似数字、一处示例与官方示例不一致、一处适用范围被写窄），另有 1 处属"合理推断但官方未逐字说明"，均不影响结论成立。
>
> 本文综合官方 `best-practices`、`sub-agents`、`memory`、`skills`、`context-window`、`agents`、`workflows`、`agent-teams`、`how-claude-code-works`、`large-codebases`、`discover-plugins`、`costs`、`goal`、`hooks`、`settings` 等页面及官方博客核对，文末附参考来源。

---

## 一、核对方法与范围

- **抓取方式**：用 curl 直接抓取官方文档的原始 markdown（`https://code.claude.com/docs/en/<页名>.md`）与官方博客 HTML，拍平换行后用 Python 逐短语比对；关键引用再单独提取上下文核对整句。
- **判定标准**：笔记里的英文 blockquote → 与原文逐字比对；中文转述 → 核对是否准确传达原文；数字、命令、路径、版本号 → 在原文中确认存在且一致。
- **覆盖范围**：`让web_fetch生效.md`、`Claude Code 提示词最佳实践.md`、`Claude Code 官方建议怎么用 subagent.md`、`Claude Code 是否建议使用 codegraph 等 RAG 工具.md`。
- **核对日期**：2026-08-04（与笔记标注的抓取日期相同，但本次是重新抓取原文独立核对）。

## 二、核对结果总览

| 笔记 | 结论 | 发现的问题 |
|---|---|---|
| `让web_fetch生效.md` | ✅ 一致（`skipWebFetchPreflight` 是官方 settings 文档记载的真实设置） | 无 |
| `Claude Code 提示词最佳实践.md` | ✅ 基本一致，逐字引用全部对上 | 1 处"官方未逐字说明"的推断 |
| `Claude Code 官方建议怎么用 subagent.md` | ✅ 基本一致，全部逐字引用对上 | 3 处次要出入 |
| `Claude Code 是否建议使用 codegraph 等 RAG 工具.md` | ✅ 一致，博客与文档引用均准确 | 无 |

---

## 三、`Claude Code 提示词最佳实践.md`

### 核对无误的重点项（抽查清单）

- **agentic 定义**：官方原文 *"Unlike a chatbot that answers questions and waits, Claude Code can read your files, run commands, make changes, and autonomously work through problems while you watch, redirect, or step away entirely."* —— 笔记中文转述准确。
- **上下文约束**：官方 *"Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."* 与 *"The context window is the most important resource to manage."* 均在。
- **验证机制 Tip**：官方 *"Give Claude a check it can run: tests, a build, a screenshot to compare."* 逐字一致。
- **探索→计划→编码**：四个阶段（Explore/Plan/Implement/Commit）、`Shift+Tab` 进入 `⏸ plan mode on`、`claude --permission-mode plan`、`Ctrl+G` 编辑计划、"If you could describe the diff in one sentence, skip the plan"、Callout 里"plan mode is useful, but also adds overhead"——全部逐字对上。
- **AskUserQuestion 采访**：官方建议提示词与笔记所引一致，含 *"Keep interviewing until we've covered everything, then write a complete spec to SPEC.md"* 及后续 *"start a fresh session to execute it"*。
- **会话管理**：`Esc`（上下文保留可重定向）、`Esc + Esc` / `/rewind`、`"Undo that"`、`/clear`、`/compact <instructions>`、`/btw`（浮层、不进入对话历史）——全部与官方一致。
- **CLAUDE.md**：`/init` 生成初版、目标 200 行以内（官方 *"target under 200 lines per CLAUDE.md file"*）、四条"何时添加"判据、存放位置表（`~/.claude/CLAUDE.md`、`./CLAUDE.md` 或 `./.claude/CLAUDE.md`、`./CLAUDE.local.md` 进 `.gitignore`）、`@README` 导入语法——全部对上。
- **自动记忆**：官方对照表 *"Loaded into | Every session | Every session (first 200 lines or 25KB)"*，笔记表格的"前 200 行或 25KB"准确。
- **CLAUDE.md 压缩偏好**：官方 `costs.md` 明说可在项目根 CLAUDE.md 写 "Compact instructions" 段，`how-claude-code-works.md` 亦说 "add a 'Compact Instructions' section to CLAUDE.md"——笔记建议成立。
- **Skills**：`disable-model-invocation: true`、"Custom commands have been merged into skills"（`.claude/commands/deploy.md` 与 `.claude/skills/deploy/SKILL.md` 都生成 `/deploy` 且行为一致）、`` !`git diff HEAD` `` 动态上下文注入——全部逐字对上。
- **常见失败模式**：kitchen sink / correcting over and over / over-specified CLAUDE.md / trust-then-verify gap / infinite exploration 五个及对应对策全部对上。

### 勘误项

**勘误 1（次要，标注性质）**：§五"用 subagent 做调研"末尾称 *"内置的 `/code-review` skill 就是用新 subagent 审查当前 diff 的 bug"*。
- 官方 `commands.md` 对 `/code-review` 的原话是 *"`/code-review` checks the diff for correctness bugs and cleanups and can apply the findings with `--fix`"*，另有 *"`/code-review <level> <pr#>` runs a multi-agent review"*——**没有一句逐字说明"用新 subagent"**。
- 该表述与官方"fresh context 对抗式复查"建议（`best-practices` 的 *"have a subagent review the diff in a fresh context"*）机制一致，属**合理推断**，不算错误；但既然笔记其余处都严格区分原话与转述，建议此处补一句"（机制推断，非官方原话）"，或直接引 commands.md 的描述。

---

## 四、`Claude Code 官方建议怎么用 subagent.md`

### 核对无误的重点项（抽查清单）

- 定义、两条使用判据（*"flood your main conversation…"*、*"Define a custom subagent when you keep spawning the same kind of worker…"*）、独立上下文架构句——逐字一致。
- 五项价值表（Preserve context / Enforce constraints / Reuse configurations / Specialize behavior / Control costs）——逐字一致。
- 上下文节省量：`context-window.md` 中 *"The subagent read 6,100 tokens of files. You got a 420-token result. That's the context savings."* 逐字一致。
- best-practices 定位：*"Since context is your fundamental constraint, subagents are one of the most powerful tools available"*、调研/验证示例、infinite exploration、"Add an adversarial review step" 的 *"Before treating a task as done, have a subagent review the diff in a fresh context and report gaps"*——全部逐字对上。
- "Choose between subagents and main conversation" 七条对照（含 *"Latency matters. Subagents start fresh and may need time to gather context"*）——逐字一致。
- 三种典型模式（isolate high-volume operations / run parallel research / chain subagents）及 Warning *"Running many subagents that each return detailed results can consume significant context"*——逐字一致。
- 选型边界：`workflows.md` *"With subagents, skills, and agent teams, Claude is the orchestrator…"*、规模 *"A few delegated tasks per turn"* vs *"Dozens to hundreds of agents per run"*、`agent-teams.md` 两句取舍原话——逐字一致。
- 后台运行版本号：**特别注意**——笔记称"自 v2.1.198 起，subagent 默认在后台运行"。这一点核对属实：`sub-agents.md` 确有 *"as of v2.1.198, subagents run in the background by default. Claude runs a subagent in the foreground when it needs the result before continuing. Background subagents run with a smaller built-in tool set…"*，`hooks.md` 也印证。最初容易被 `sub-agents.md` 里另一处 *"as of v2.1.198, Explore inherits the main conversation's model…"* 混淆，但两处并存、笔记引用的是前者，正确。
- 定义 subagent 四条 best practices、优先级表（Managed settings > `--agents` > 项目 `.claude/agents/` > 用户 `~/.claude/agents/` > 插件）、三种调用方式（自然语言 / @-mention / `--agent`+`agent` setting）、内置 Explore/Plan/general-purpose 描述——全部对上。

### 勘误项

**勘误 2（数字近似）**：§二把节省量写成"大约 **1/15**"。
- 按笔记自己引的数字：420 ÷ 6100 ≈ **6.9% ≈ 1/14.5**，不是 1/15。1/15 = 6.67%，偏小一点。
- 影响很小（原文带"大约"），但既然是与官方数字配套的换算，建议改为"约 **1/14.5**（约 6.9%）"，或保留"约 1/15"并在脚注给精确值。

**勘误 3（示例与官方示例不一致）**：§七的自定义 subagent frontmatter 示例。

| 字段 | 笔记示例 | 官方 `sub-agents.md` 的 code-reviewer 示例 |
|---|---|---|
| `description` | `Reviews code for quality and best practices` | `Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.` |
| `tools` | `Read, Glob, Grep` | `Read, Grep, Glob, Bash` |
| `model` | `sonnet` | `inherit` |

- 三点值在语法上都合法（`model` 字段官方支持 `sonnet`/`opus`/`haiku`/`fable`/完整模型 ID/`inherit`），所以不是"非法配置"，而是**与官方示例不一致**。若该代码块意图是"复现官方格式给读者照抄"，建议改成官方原例（尤其 `tools` 含 `Bash`、`model: inherit`）；若只是自拟示意，建议在代码块前注明"示例为自拟，非官方原文"。

**勘误 4（适用范围被写窄）**：§八 Explore 内置条目写"跳过 CLAUDE.md 与 git status 以保持快速低成本"。
- 官方 `sub-agents.md` 原话是：*"**Explore and Plan** skip your CLAUDE.md files and the parent session's git status to keep research fast and inexpensive."* —— **Explore 和 Plan 两者都**跳过，不止 Explore。
- 建议把该句挪到表格上方或同时标注 Plan，避免读者误以为只有 Explore 有此行为。

---

## 五、`Claude Code 是否建议使用 codegraph 等 RAG 工具.md`

### 核对结论：未发现与官方不一致处

- **博客引用**：*"RAG-powered AI coding tools work by embedding the entire codebase…"*、*"Claude Code navigates a codebase the way a software engineer would…"*、*"At large scale, those systems can fail because embedding pipelines can't keep up…"*、*"By the time a developer queries the index…"*、*"Retrieval then returns a function the team renamed two weeks ago…"*、*"Agentic search avoids those failure modes. There's no embedding pipeline or centralized index…"*、*"doesn't require a codebase index to be built, maintained, or uploaded to a server"*、*"layering context with CLAUDE.md files and skills"*、*"Claude loads them additively as it moves through the codebase"*、*"some teams spin up a read-only subagent to map a subsystem…"*——**全部逐字核对无误**，且出处（博客 vs 文档）标注正确。
- **工具分类**：`how-claude-code-works.md` 的五类（File operations / Search / Execution / Web / Code intelligence）与 Code intelligence 描述 *"See type errors and warnings after edits, jump to definitions, find references (requires code intelligence plugins)"*——逐字一致。
- **LSP**：*"In a large codebase, finding where a symbol is defined or used can cost many file reads and grep calls. Code intelligence plugins connect Claude to a language server…"*（出自 `large-codebases.md`）与 *"LSP returns only the references that point to the same symbol, so the filtering happens before Claude reads anything"*（出自官方博客）——都准确，且**出处归属正确**。
- **MCP 接 RAG**：`large-codebases.md` *"MCP servers: if your organization already runs a code search or RAG index over the repository, expose it as an MCP tool so Claude queries it instead of reading files directly"*——逐字一致。
- **插件清单**：C/C++、C#、Go、Java、Kotlin、Lua、PHP、Python、Rust、Swift、TypeScript 十一种语言在 `discover-plugins.md` 的 LSP 插件表中全部存在；`/plugin install typescript-lsp@claude-plugins-official` 亦为文档原例。
- **"文档没有 RAG 专页"的结论**：本次重扫 `llms.txt`（180 行，页面链接约 178 个），确实无任何 RAG / embedding / 向量索引专页——"约 180 个页面"的表述合理。

---

## 六、`让web_fetch生效.md`

- `skipWebFetchPreflight`：**确认是官方设置**。`settings.md` 原文：*"Skip the WebFetch domain safety check that sends each requested hostname to api.anthropic.com before fetching. Set to true in environments that block traffic to Anthropic…"*。笔记描述（配 `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` 解决代理环境 web_fetch 失败）是本地环境配置，无官方口径可对照，无勘误。

---

## 七、勘误汇总表

| # | 位置 | 原表述 | 官方依据 | 建议 |
|---|---|---|---|---|
| 1 | `提示词最佳实践.md` §五 | "内置的 `/code-review` skill 就是用新 subagent 审查当前 diff 的 bug" | `commands.md` 只写 *"checks the diff for correctness bugs and cleanups…"*，未逐字说"用新 subagent" | 标注为"机制推断"，或改引 commands.md 原话 |
| 2 | `subagent.md` §二 | "大约 1/15" | 420 ÷ 6100 ≈ 6.9% ≈ **1/14.5** | 改为"约 1/14.5（约 6.9%）" |
| 3 | `subagent.md` §七 | frontmatter 示例 `tools: Read, Glob, Grep`、`model: sonnet` | 官方示例 `tools: Read, Grep, Glob, Bash`、`model: inherit` | 改成官方原例，或注明"示例为自拟" |
| 4 | `subagent.md` §八 | 仅 Explore "跳过 CLAUDE.md 与 git status" | 官方原文是 **Explore and Plan** 都跳过 | 补上 Plan |

> 说明：以上 4 项均不影响各篇的核心结论与"官方推荐/不推荐"的判断。其余逐字引用、数字、命令、路径、版本号经核对全部准确。

---

## 参考来源

本次勘误基于 2026-08-04 重新抓取的官方原文逐字比对（`code.claude.com/docs/en/*.md` 原始 markdown + 官方博客 HTML），与被核对笔记的抓取日期相同但为独立抓取：

- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices.md
- **Create custom subagents** — https://code.claude.com/docs/en/sub-agents.md
- **How Claude remembers your project** — https://code.claude.com/docs/en/memory.md
- **Extend Claude with skills** — https://code.claude.com/docs/en/skills.md
- **Explore the context window** — https://code.claude.com/docs/en/context-window.md
- **Run agents in parallel** — https://code.claude.com/docs/en/agents.md
- **Orchestrate subagents at scale with dynamic workflows** — https://code.claude.com/docs/en/workflows.md
- **Orchestrate teams of Claude Code sessions** — https://code.claude.com/docs/en/agent-teams.md
- **How Claude Code works** — https://code.claude.com/docs/en/how-claude-code-works.md
- **Set up Claude Code in a monorepo or large codebase** — https://code.claude.com/docs/en/large-codebases.md
- **Discover and install prebuilt plugins** — https://code.claude.com/docs/en/discover-plugins.md
- **Manage costs effectively** — https://code.claude.com/docs/en/costs.md
- **Keep Claude working toward a goal** — https://code.claude.com/docs/en/goal.md
- **Hooks reference** — https://code.claude.com/docs/en/hooks.md
- **Claude Code settings** — https://code.claude.com/docs/en/settings.md
- **Explore the .claude directory** — https://code.claude.com/docs/en/claude-directory.md
- **Commands** — https://code.claude.com/docs/en/commands.md
- **官方博客：How Claude Code works in large codebases** — https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`、`Agent/ClaudeCode/让web_fetch生效.md`（均为本文核对对象）。调研流程见技能 `claude-code-best-practices`。
