# Agent/ClaudeCode 笔记勘误：与 Claude Code 官方文档的核对结果

> **一句话总结**：核对 `Agent/ClaudeCode/` 下 4 篇笔记与 Claude Code 官方文档（`code.claude.com/docs` 全站 + `claude.com/blog` 官方博客，均于 2026-08-04 通过 web_fetch 获取并逐字比对）后，**整体结论是"基本准确"**——绝大多数官方原话引用都与原文一致，未发现任何"结论性错误"。共发现 **3 处需要修正**，且都很轻微：**两处**出在 RAG 笔记的官方博客引用上（**一处分句语序被颠倒、一处引文被截断且标点被改动**），**一处**是提示词最佳实践笔记里的产品名错别字（"CLAUDE Code" 应为 "Claude Code"）。subagent 笔记与 web 配置笔记未发现与官方不一致的说法。

---

## 一、核对范围与方法

本次核对覆盖 `Agent/ClaudeCode/` 下全部 4 篇笔记：

| 笔记 | 核对依据 |
|---|---|
| `Claude Code 是否建议使用 codegraph 等 RAG 工具.md` | 官方博客 + `how-claude-code-works`、`large-codebases`、`discover-plugins`、`context-window`、`best-practices`、`costs` |
| `Claude Code 提示词最佳实践.md` | `best-practices`、`memory`、`skills`、`commands` |
| `Claude Code 官方建议怎么用 subagent.md` | `sub-agents`、`best-practices`、`context-window`、`agents`、`workflows`、`agent-teams` |
| `让web_fetch生效.md` | 本机环境配置说明，不含官方立场表述，仅检查事实性描述 |

方法：先抓 `https://code.claude.com/docs/llms.txt` 全站索引确认所有引用 URL 均存在（索引含 174 个文档 URL），再逐页抓取原文，把笔记里每一处加引号的"官方原话"与原文**逐字**比对。下方"问题"仅列有出入处；"核对详情"列已确认无误的部分。

---

## 二、发现的问题（共 3 处）

### 问题 1：RAG 笔记 —— 博客引文分句语序被颠倒

**位置**：`Claude Code 是否建议使用 codegraph 等 RAG 工具.md` 第 61–63 行

**笔记现在引用为**：

> "…doesn't require a codebase index to be built, maintained, or uploaded to a server. It operates locally on the developer's machine."

**官方博客原文是**（`https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start`）：

> "It operates locally on the developer's machine and doesn't require a codebase index to be built, maintained, or uploaded to a server."

**差异**：两个分句的顺序被**颠倒**了——笔记把 "doesn't require a codebase index…" 放前面、"operates locally…" 放后面；官方是相反顺序（"operates locally" 在前、"doesn't require" 在后）。两句的**含义没变**，但既然标了引号当作"官方原话"，语序颠倒就是引用不准确。

**建议改为**：

> "It operates locally on the developer's machine and doesn't require a codebase index to be built, maintained, or uploaded to a server."
> （它在开发者本机上本地运行，不需要构建、维护、或上传一个代码库索引到服务器。）

---

### 问题 2：RAG 笔记 —— 博客引文被截断、标点被改动

**位置**：`Claude Code 是否建议使用 codegraph 等 RAG 工具.md` 第 90 行

**笔记现在引用为**：

> "Surfacing this to Claude gives it symbol-level precision: it can follow a function call to its definition, trace references across files."

**官方博客原文是**（LSP 一节）：

> "Surfacing this to Claude gives it symbol-level precision," … "trace references across files, and distinguish between identically named functions in different languages."

**差异**：两处。① 官方原文 "symbol-level precision," 后是**逗号**，笔记改成了**冒号**；② 原句并未在 "trace references across files" 处结束，而是继续到 "and distinguish between identically named functions in different languages"（还能区分不同语言里的同名函数），笔记用句号截断，让引文看起来完整了。

**建议改为**（补全 + 恢复逗号）：

> "Surfacing this to Claude gives it symbol-level precision," … "trace references across files, and distinguish between identically named functions in different languages."
> （把这套能力暴露给 Claude，它就有了**符号级精度**：能顺着一个函数调用追到定义、跨文件追踪引用，还能区分不同语言里的同名函数。）

> 若想保留精简版，建议把"引号内原话"改为转述，例如：官方博客指出，这给了 Claude **符号级精度**——能顺着函数调用追到定义、跨文件追踪引用。

---

### 问题 3：提示词最佳实践笔记 —— 产品名错别字

**位置**：`Claude Code 提示词最佳实践.md` 第 123 行

**笔记现在写为**：

> "CLAUDE Code 内置的 `/code-review` skill 就是用新 subagent 审查当前 diff 的 bug。"

**差异**：产品名写成了全大写的 "CLAUDE Code"，官方写法是 "**Claude Code**"。这一处是错别字，非事实性错误。

**附带核对**：句中的事实断言本身是**正确**的——官方命令文档确认 `/code-review` 是**内置（bundled）skill**："Review the current diff for correctness bugs and cleanup opportunities"；`best-practices` 也写明 "run the bundled `/code-review` skill, which reviews the current diff for bugs in a fresh subagent and returns findings to the session"。所以只需改错别字：

**建议改为**：

> "Claude Code 内置的 `/code-review` skill 就是用新 subagent 审查当前 diff 的 bug。"

---

## 三、逐篇核对详情（确认无误的部分）

### 1. `Claude Code 是否建议使用 codegraph 等 RAG 工具.md`

除上述问题 1、2 两处博客引文外，其余**全部核实无误**：

- **博客核心引文 6 条逐字一致**：RAG 工作方式（"RAG-powered AI coding tools work by embedding the entire codebase…"）、Claude Code 导航方式（"navigates a codebase the way a software engineer would…"）、embedding 流水线失效（"embedding pipelines can't keep up with active engineering teams"）、索引滞后（"reflects the codebase as it previously existed…"）、过时检索（"returns a function the team renamed two weeks ago…"）、agentic search 对比（"There's no embedding pipeline or centralized index…"）。
- **博客其余引文**："Claude loads them additively as it moves through the codebase." ✓、"root file for the big picture, subdirectory files for local conventions" ✓、"LSP returns only the references that point to the same symbol, so the filtering happens before Claude reads anything." ✓、subagent 例子（"spin up a read-only subagent to map a subsystem…"）✓。
- **`large-codebases` 文档引文**：LSP 引文（"Code intelligence plugins connect Claude to a language server…"）✓、全文唯一提到 RAG 的 MCP 建议（"if your organization already runs a code search or RAG index…"）✓。
- **`how-claude-code-works` 五类工具表**：文件操作 / 搜索 / 执行 / Web / 代码智能五类与官方原文一致；"搜索"（"Find files by pattern, search content with regex, explore codebases"）与"代码智能"（"See type errors and warnings after edits, jump to definitions, find references (requires code intelligence plugins)"）类别描述吻合。
- **`discover-plugins`**：LSP 插件清单的 11 种语言（C/C++、C#、Go、Java、Kotlin、Lua、PHP、Python、Rust、Swift、TypeScript）与官方表逐一对应；安装语法 `/plugin install <语言>-lsp@claude-plugins-official` 与官方 `/plugin install <name>@claude-plugins-official` 一致；"These operations give Claude more precise navigation than grep-based search" ✓；"需要先装语言服务器二进制" ✓。
- **`best-practices`**："The context window is the most important resource to manage" ✓。

### 2. `Claude Code 提示词最佳实践.md`

除问题 3 的错别字外，**全部核实无误**：

- **memory 文档的 CLAUDE.md vs 自动记忆对照表**：谁写（You / Claude）、装什么（Instructions and rules / Learnings and patterns）、范围（Project, user, or org / **Per repository, shared across worktrees**——笔记写"按仓库" ✓）、何时加载（Every session / **first 200 lines or 25KB** ✓）、用途（Build commands, debugging insights, preferences Claude discovers ✓）——与官方表格逐项一致。
- **best-practices**：验证机制引文（"Give Claude a check it can run: tests, a build, a screenshot to compare…"）、探索→计划→编码四阶段、具体上下文四策略、会话管理、"Would removing this cause Claude to make mistakes?" 精简判据、常见失败模式五条，均与原文一致。
- **`/goal`**：笔记称"把检查设为 `/goal` 条件，独立评估器每轮后重查，Claude 持续工作直到条件成立"——官方命令文档确认 `/goal` 存在（"Set a goal: Claude keeps working across turns until the condition is met"），best-practices 原文正是 "A separate evaluator re-checks it after every turn" ✓。
- **`/btw`**（答案不进对话历史）✓、**`/code-review`**（见问题 3）✓。
- **skills 文档**：自定义命令并入 skills（`.claude/commands/deploy.md` 与 `.claude/skills/deploy/SKILL.md` 都会生成 `/deploy`）、`!` 动态上下文注入（`` !`git diff HEAD` ``）、`disable-model-invocation: true`、description 决定自动加载，均与官方原文一致。

### 3. `Claude Code 官方建议怎么用 subagent.md`

**未发现与官方不一致的说法**，重点条目均逐字核对：

- subagent 定位与使用判据（"a side task would flood your main conversation…"）、独立上下文窗口与权限。
- 上下文节省量示例（subagent 读 **6,100 token**、主对话只收到 **420 token**，"That's the context savings"）。
- best-practices 定位（"subagents are one of the most powerful tools available"）、调研 / 验证 / 对抗式复查用法、"无限探索"失败模式。
- 主对话 vs subagent 对照表（迭代/共享上下文/延迟 → 主对话；大输出/权限限制/自成一体 → subagent）。
- 三种典型模式（隔离高流量、并行研究、链式）及"省过程不省结果"告诫。
- subagents / agent view / agent teams / dynamic workflows 四方案对照、规模数字、agent teams 协调开销。
- 内置 subagent（Explore / Plan / general-purpose）、后台运行（v2.1.198+）、嵌套三层、配额（200/会话、20 并发）、`/btw` 边界。

### 4. `让web_fetch生效.md`

纯环境配置说明（代理与 `skipWebFetchPreflight`），不涉及官方文档立场，无勘误项。

---

## 四、修正建议汇总表

| 位置 | 类型 | 现状 | 官方原文 | 建议 |
|---|---|---|---|---|
| RAG 笔记 L61–63 | 引文语序颠倒 | "…doesn't require a codebase index… It operates locally…" | "It operates locally… and doesn't require a codebase index…" | 按官方语序重排分句 |
| RAG 笔记 L90 | 引文截断 + 标点改动 | "symbol-level precision: … trace references across files." | "symbol-level precision," … "trace references across files, and distinguish between identically named functions in different languages." | 补全到 "…different languages" 或改为转述 |
| 提示词笔记 L123 | 产品名错别字 | "CLAUDE Code 内置的 `/code-review` skill…" | "Claude Code"（`/code-review` 确为 bundled skill） | "CLAUDE" → "Claude" |

---

## 五、参考来源

本次勘误核对所用官方资料（均于 2026-08-04 通过 web_fetch 获取）：

- **文档索引** — https://code.claude.com/docs/llms.txt （确认 174 个文档 URL，全部引用页均存在）
- **官方博客：How Claude Code works in large codebases** — https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start （RAG vs agentic search 全部引文；问题 1、2 即出自此篇）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices （验证机制、探索→计划→编码、CLAUDE.md 精简判据、失败模式、"context window is the most important resource"）
- **Memory: How Claude remembers your project** — https://code.claude.com/docs/en/memory （CLAUDE.md vs 自动记忆对照表、"按仓库" scope、"前 200 行或 25KB"）
- **Skills: Extend Claude with skills** — https://code.claude.com/docs/en/skills （自定义命令并入 skills、`!` 注入、disable-model-invocation）
- **How Claude Code works** — https://code.claude.com/docs/en/how-claude-code-works （五类内置工具表）
- **Set up Claude Code in a monorepo or large codebase** — https://code.claude.com/docs/en/large-codebases （LSP 引文、MCP RAG 建议）
- **Discover and install prebuilt plugins** — https://code.claude.com/docs/en/discover-plugins （LSP 插件 11 语言清单与安装语法）
- **Commands** — https://code.claude.com/docs/en/commands （`/code-review`、`/goal`、`/btw` 均为内置命令）
- **Create custom subagents / Explore the context window / Run agents in parallel / Dynamic workflows / Agent teams** — https://code.claude.com/docs/en/sub-agents · /context-window · /agents · /workflows · /agent-teams （subagent 笔记全部引用）

> 相关文档：`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`、`Agent/ClaudeCode/让web_fetch生效.md`。
