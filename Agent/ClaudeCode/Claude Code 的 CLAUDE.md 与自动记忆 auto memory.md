# Claude Code 的 CLAUDE.md 与自动记忆 auto memory

> **一句话总结**：Claude Code 有两套**互补**的记忆系统，官方划分的核心轴是**"谁写的"**——**CLAUDE.md 是你（人）写的持久指令，auto memory 是 Claude 写给自己的笔记**。CLAUDE.md 用来**引导 Claude 的行为**（编码规范、工作流、项目架构），按 project / user / org 分层、走版本控制；auto memory 用来**让 Claude 从你的纠正和偏好里自动学习**（构建命令、调试洞见、它发现的偏好），按 git 仓库存储在机器本地、Claude 自己决定记什么。两者都在**每次会话开始时加载**，且都被当作**上下文（context）而非强制配置（enforced configuration）**。
>
> 本文基于 Claude Code 官方文档 `How Claude remembers your project`、`Explore the .claude directory`、`Explore the context window`、`How Claude Code uses prompt caching`、`Extend Claude Code` 与 `Glossary` 整理，文末附参考来源。

Claude Code 的每个会话都从一个全新的上下文窗口开始。跨会话的知识，靠两套机制带过来。官方文档 `How Claude remembers your project` 开篇就给出这个二分：

> "Two mechanisms carry knowledge across sessions:
> * **CLAUDE.md files**: instructions you write to give Claude persistent context
> * **Auto memory**: notes Claude writes itself based on your corrections and preferences"
> （两套机制把知识带过会话边界：**CLAUDE.md 文件**——你写的、给 Claude 持久上下文的指令；**auto memory**——Claude 根据你的纠正和偏好自己写的笔记。）

这两句话里的两个动词——"**you write**"和"**writes itself**"——就是整个区分的起点。

---

## 一、官方给出的第一张对照表

`How Claude remembers your project` 页有一张现成的对比表，是理解整套设计的锚点：

| 维度 | CLAUDE.md files | Auto memory |
| :--- | :--- | :--- |
| **谁写的**（Who writes it） | 你（You） | Claude |
| **装了什么**（What it contains） | 指令与规则（Instructions and rules） | 学习与模式（Learnings and patterns） |
| **范围**（Scope） | 项目、用户或组织（Project, user, or org） | 每个仓库一份，跨 worktrees 共享（Per repository, shared across worktrees） |
| **加载进**（Loaded into） | 每个会话（Every session） | 每个会话（前 200 行或 25KB） |
| **用来做什么**（Use for） | 编码规范、工作流、项目架构（Coding standards, workflows, project architecture） | 构建命令、调试洞见、Claude 发现的偏好（Build commands, debugging insights, preferences Claude discovers） |

表下官方用一句话点出两者的分工：

> "Use CLAUDE.md files when you want to guide Claude's behavior. Auto memory lets Claude learn from your corrections without manual effort."
> （想**引导 Claude 的行为**，用 CLAUDE.md 文件；想让 Claude**从你的纠正中自动学习**、不用你手动费力，靠 auto memory。）

`Glossary` 里给 auto memory 下定义时，更是直接把它定位成 CLAUDE.md 的"Claude 写的另一半"：

> "Auto memory is the Claude-written counterpart to CLAUDE.md, which you write."
> （auto memory 是 CLAUDE.md 的"Claude 写"的对应物——而 CLAUDE.md 是你写的。）

## 二、CLAUDE.md：你写的持久指令（引导行为）

### 定义

> "CLAUDE.md files are markdown files that give Claude persistent instructions for a project, your personal workflow, or your entire organization. You write these files in plain text; Claude reads them at the start of every session."
> （CLAUDE.md 是给 Claude 提供持久指令的 markdown 文件，覆盖一个项目、你的个人工作流、或整个组织。你用纯文本写这些文件，Claude 在每个会话开始时读取。）

### 放在哪里：四层作用域（按加载顺序）

| 作用域 | 位置 | 用途 | 共享给谁 |
| :--- | :--- | :--- | :--- |
| **Managed policy**（组织托管） | Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`（macOS/Linux 各有对应路径） | 组织级指令，由 IT/DevOps 管理 | 组织内所有用户 |
| **User instructions**（用户级） | `~/.claude/CLAUDE.md` | 你的个人偏好，作用于所有项目 | 只有你（跨项目） |
| **Project instructions**（项目级） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享的项目约定 | 团队成员（走版本控制） |
| **Local instructions**（本地级） | `./CLAUDE.local.md` | 你个人的项目专属偏好，记得加进 `.gitignore` | 只有你（当前项目） |

### 什么时候该往里写

> "Treat CLAUDE.md as the place you write down what you'd otherwise re-explain."
> （把 CLAUDE.md 当成"那些你本来会反复重讲的东西"的存放处。）

官方列出的"该加"信号：

- Claude 第二次犯同一个错误
- 一次 code review 抓到了 Claude 本应了解的代码库知识
- 你发现同一个纠正/澄清，上个会话刚打过一遍
- 一个新同事要上手，需要同样的上下文才能高效工作

`Extend Claude Code` 页给了一条很好记的触发规则：

> "Claude gets a convention or command wrong twice → Add it to CLAUDE.md"
> （Claude 把一个约定或命令搞错两次 → 把它加进 CLAUDE.md。）

### 加载机制

- **启动时全量加载**：Claude Code 从工作目录向上遍历目录树，检查每一级有没有 `CLAUDE.md` / `CLAUDE.local.md`，把所有发现的文件**拼接进上下文，而不是互相覆盖**；顺序是从文件系统根到工作目录，越靠近启动目录的指令读得越晚。
- **子目录按需加载**：工作目录以下的子目录里的 CLAUDE.md，不在启动时加载，而是当 Claude 读取该子目录里的文件时才加载。
- **建议 ≤ 200 行**：`Target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence.`（每份 CLAUDE.md 目标控制在 200 行以内。更长的文件更吃上下文、服从度也更低。）注意：CLAUDE.md 超长**仍会全量加载**——它没有 auto memory 那样的 200 行/25KB 截断。
- **它是"用户消息"而非"系统提示"**：官方排障章节明说——

> "CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there's no guarantee of strict compliance."
> （CLAUDE.md 内容是跟在系统提示之后、以**用户消息**形式送达的，不是系统提示本身的一部分。Claude 会读它、尽量遵守它，但并不保证严格服从。）

## 三、auto memory：Claude 写给自己的笔记（自动学习）

### 定义

> "Auto memory lets Claude accumulate knowledge across sessions without you writing anything. Claude saves notes for itself as it works: build commands, debugging insights, architecture notes, code style preferences, and workflow habits. Claude doesn't save something every session. It decides what's worth remembering based on whether the information would be useful in a future conversation."
> （auto memory 让你**一行都不用写**，Claude 就能跨会话积累知识。Claude 边干活边给自己存笔记：构建命令、调试洞见、架构笔记、代码风格偏好、工作流习惯。它不是每个会话都存。它根据"这条信息在未来对话里有没有用"来决定什么值得记。）

注意这句话的落点：**记什么是 Claude 自己判断的**，判据是"未来对话是否有用"。这就是它和 CLAUDE.md（你决定记什么）最本质的分工。

### 存在哪里

> "Each project gets its own memory directory at `~/.claude/projects/<project>/memory/`. The `<project>` path is derived from the git repository, so all worktrees and subdirectories within the same repo share one auto memory directory."
> （每个项目有自己的记忆目录 `~/.claude/projects/<project>/memory/`。`<project>` 由 git 仓库派生，所以同一仓库的所有 worktrees 和子目录共享同一个 auto memory 目录。）

目录结构（官方示例）：

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md          # Concise index, loaded into every session
├── debugging.md       # Detailed notes on debugging patterns
├── api-conventions.md # API design decisions
└── ...                # Any other topic files Claude creates
```

> "MEMORY.md acts as an index of the memory directory."
> （MEMORY.md 是记忆目录的索引。）

一个关键特性：**auto memory 是机器本地的**（machine-local），不随 git 共享、不上传云端。与之对比，项目级 CLAUDE.md 恰恰是靠版本控制共享给团队的。

### 怎么加载

- **启动只读索引**：每个会话只加载 `MEMORY.md` 的**前 200 行或前 25KB**（先到者为准）；超出的部分启动时不加载。
- **主题文件按需读**：`debugging.md`、`patterns.md` 这类主题文件**不在启动时加载**，Claude 需要时用标准文件工具去读。
- **会话中持续读写**：Claude 在会话中会读写记忆文件。当你看到界面里冒出 "Saved 2 memories" 或 "Recalled 2 memories"，就是它在更新/读取 `~/.claude/projects/<project>/memory/`。
- **与 CLAUDE.md 的加载差异，官方明说**：

> "This limit applies only to `MEMORY.md`. CLAUDE.md files are loaded in full regardless of length."
> （这个 200 行/25KB 限制只适用于 `MEMORY.md`。CLAUDE.md 无论多长都是全量加载。）

### 开关与审计

- 默认**开启**；可在会话里 `/memory` 用 toggle 关，或写 `autoMemoryEnabled: false`，或环境变量 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。
- 存到别处：`autoMemoryDirectory` 设置可换目录。
- **审计**：所有 auto memory 文件都是纯 markdown，随时可读可改可删。`/memory` 命令可浏览/打开记忆文件夹；`/context` 查看当前会话实际加载了哪些。

## 四、一张表讲清全部关键差异

把前面几节的细节汇总成一张更完整的对照：

| 维度 | CLAUDE.md | Auto memory |
| :--- | :--- | :--- |
| **写入者** | 你（人） | Claude（模型） |
| **内容** | 指令与规则 | 学习与模式 |
| **典型用途** | 编码规范、工作流、项目架构、"always do X" | 构建命令、调试洞见、Claude 发现的偏好 |
| **谁决定记什么** | 你决定 | Claude 判断"未来对话是否有用" |
| **存储位置** | `./CLAUDE.md`、`.claude/CLAUDE.md`、`~/.claude/CLAUDE.md`、`CLAUDE.local.md`、managed 位置 | `~/.claude/projects/<project>/memory/`（按 git 仓库） |
| **范围** | 项目 / 用户 / 组织 | 每仓库一份，同仓库 worktrees 共享 |
| **是否共享** | 项目级走版本控制、团队共享 | 机器本地，不跨机器、不上云 |
| **启动加载** | 全量加载（向上遍历目录树、多文件拼接） | 只加载 `MEMORY.md` 前 200 行 / 25KB |
| **其余内容** | 子目录里的按需加载 | 主题文件按需读 |
| **写入时机** | 会话外你手动写，或让 Claude "add this to CLAUDE.md" | 会话中 Claude 自动写（"Saved 2 memories"） |
| **compaction 后** | 项目根 CLAUDE.md 从磁盘重读 | 从磁盘重读 |
| **会话中编辑** | 启动即读入，mid-session 编辑不生效（下次 /clear、/compact、重启才生效） | Claude 会话中持续读写 |

## 五、实际使用：想"记下来"时，官方建议怎么分流

官方给了非常具体的操作指引——同样是"让 Claude 记住"，说辞不同，落点就不同：

> "When you ask Claude to remember something, like 'always use pnpm, not npm' or 'remember that the API tests require a local Redis instance,' Claude saves it to auto memory. To add instructions to CLAUDE.md instead, ask Claude directly, like 'add this to CLAUDE.md,' or edit the file yourself via `/memory`."
> （当你让 Claude 记住某事——比如"永远用 pnpm 别用 npm"、"记住 API 测试要本地 Redis"——Claude 会把它存进 **auto memory**。如果你想把指令加进 **CLAUDE.md**，就直接说"add this to CLAUDE.md"，或者自己用 `/memory` 去编辑文件。）

可执行的分流规则：

- **"记住这件事"**（口语化的记忆请求）→ auto memory，Claude 自己判断怎么存。
- **"把这条加进 CLAUDE.md"**（明确的指令请求）→ CLAUDE.md，由你/版本控制管理。
- **命令差异**：`/memory` 列出并编辑 CLAUDE.md、CLAUDE.local.md 与 auto memory 文件、开关 auto memory；`/context` 只用来**核对当前会话实际加载了什么**（排障首选）。

另外，官方强调"同一条内容只进一个地方"——重复纠正应该沉淀成 CLAUDE.md 的持久指令，而不是每次在对话里再纠正一次：

> "A repeated mistake or a recurring review comment is a CLAUDE.md edit, not a one-off correction in chat."
> （重复犯的错、反复出现的 review 意见，是一次 CLAUDE.md 编辑，而不是聊天里的一次性纠正。）

## 六、容易踩的坑

1. **两者都不是"强制配置"**。官方原话："Claude treats them as context, not enforced configuration."（Claude 把它们当上下文，而不是强制执行配置。）想**不管 Claude 怎么想都拦住某个动作**，要用 `PreToolUse` hook，而不是写进 CLAUDE.md 或 auto memory。
2. **CLAUDE.md 没有 200 行的硬性截断**，但超过 200 行会消耗更多上下文、降低服从度；而 `MEMORY.md` 有**硬截断**（200 行 / 25KB），超出的部分下次加载直接被丢弃，Claude Code 会报错提醒重写索引。
3. **mid-session 改 CLAUDE.md 不生效**。官方 `prompt caching` 文档：项目根与用户级 CLAUDE.md 在会话开始时一次性读入并常驻内存，会话中编辑既不清缓存、也不生效，要等下次 `/clear`、`/compact` 或重启。
4. **compaction 的存活差异**。`context-window` 文档："Project-root CLAUDE.md and unscoped rules — Re-injected from disk"；"Auto memory — Re-injected from disk"；但 "Nested CLAUDE.md in subdirectories — Lost until a file in that subdirectory is read again"。即：项目根 CLAUDE.md 和 auto memory 都能扛过 `/compact`，**嵌套在子目录里的 CLAUDE.md 不会自动重新注入**。
5. **auto memory 不跨机器**。换机器/云端环境就没有它；CLAUDE.md 因走版本控制而能跟着仓库走。
6. **subagent 的记忆是第三套**。主会话的 auto memory 不会加载进 subagent（fork 除外）；subagent 可用 frontmatter `memory:` 启用自己独立的记忆目录（`.claude/agent-memory/` 等）。别把它和主会话 auto memory 混为一谈。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 获取；web_fetch 触发 `ECONNRESET`，改经本地代理 127.0.0.1:7890 用 curl 抓取原文）：

- **How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （核心页：CLAUDE.md vs auto memory 对照表、"Who writes it" 二分、CLAUDE.md 四层作用域与加载机制、auto memory 存储与 200 行/25KB 加载限制、`/memory` 与 `/context` 用法、排障与 compaction 行为）
- **Explore the .claude directory** — https://code.claude.com/docs/en/claude-directory
  （`.claude/` 目录结构总览：CLAUDE.md、settings、hooks、skills、rules、agent-memory 各司其职；"commit project files to git to share" 与 `~/.claude` 个人配置的分界）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （启动时加载项清单："CLAUDE.md、auto memory、MCP 工具名、skill 描述都加载进上下文"；compaction 后什么存活、什么丢失）
- **How Claude Code uses prompt caching** — https://code.claude.com/docs/en/prompt-caching
  （"Editing CLAUDE.md mid-session"：CLAUDE.md 常驻内存、编辑不生效的缓存机制）
- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （"Claude 搞错两次 → 加进 CLAUDE.md"的触发规则；CLAUDE.md vs Skill vs Rules vs Hooks 的分工）
- **Glossary** — https://code.claude.com/docs/en/glossary
  （CLAUDE.md 与 auto memory 的官方定义；auto memory 是 "the Claude-written counterpart to CLAUDE.md" 的原话出处）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口管理）、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 及其独立记忆）、`Agent/ClaudeCode/让web_fetch生效.md`（抓取环境的代理配置）。
