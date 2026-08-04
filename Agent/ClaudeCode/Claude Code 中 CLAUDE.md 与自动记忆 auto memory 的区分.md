# Claude Code 中 CLAUDE.md 与自动记忆（auto memory）的区分

> **一句话总结**：Claude Code 有**两条互补的跨会话记忆机制**——**CLAUDE.md 是你（人）写的持久指令**（编码规范、工作流、项目架构），**auto memory 是 Claude 自己写的笔记**（构建命令、调试心得、它发现你的偏好）；两者都在**每个会话开始时加载进上下文**，但官方强调**它们只是"上下文"而非"强制配置"**，作者、内容、作用域、可控性完全不同——想分清"该让 Claude 记什么"还是"该自己写进 CLAUDE.md"，只需记住一句话：**想让 Claude 自动记住 → 它记进 auto memory；想给它持久指令 → 写进 CLAUDE.md。**
>
> 本文基于 Claude Code 官方文档 `How Claude remembers your project (memory)`、`Explore the .claude directory`、`Glossary`、`How Claude Code uses prompt caching`、`Explore the context window` 整理，文末附参考来源。第六节为 2026-08-04 重抓官方文档核验时补充的新增细节。

Claude Code 的每个会话都从**全新的上下文窗口**开始（"Each Claude Code session begins with a fresh context window"），上一次会话的记忆不会天然带过来。官方文档用一句话点出两条跨会话机制的定位：

> "Give Claude persistent instructions with CLAUDE.md files, and let Claude accumulate learnings automatically with auto memory."
> （用 CLAUDE.md 文件给 Claude 持久指令，用自动记忆让 Claude 自动积累学到的经验。）

---

## 一、两条机制的总览对比（官方原表）

官方文档 `memory.md` 开头就给出了一张直接对比表，是全站对二者区别最凝练的表述：

| 维度 | **CLAUDE.md files** | **Auto memory** |
|---|---|---|
| **Who writes it（谁写）** | You（你） | Claude |
| **What it contains（内容）** | Instructions and rules（指令与规则） | Learnings and patterns（学到的经验与模式） |
| **Scope（作用域）** | Project, user, or org（项目 / 用户 / 组织） | Per repository, shared across worktrees（按仓库，跨 worktree 共享） |
| **Loaded into（加载进）** | Every session（每个会话） | Every session（每个会话，前 200 行或 25KB） |
| **Use for（用途）** | Coding standards, workflows, project architecture（编码规范、工作流、项目架构） | Build commands, debugging insights, preferences Claude discovers（构建命令、调试心得、Claude 发现的偏好） |

官方给出的使用准则：

> "Use CLAUDE.md files when you want to guide Claude's behavior. Auto memory lets Claude learn from your corrections without manual effort."
> （想**引导** Claude 的行为就用 CLAUDE.md；想让它**从你的纠正中自动学习**、不用你动手，就靠 auto memory。）

`Glossary` 里给 auto memory 下定义时，更是直接把它定位成 CLAUDE.md 的"Claude 写的另一半"——一句话道尽区分核心：

> "Auto memory is the Claude-written counterpart to CLAUDE.md, which you write."
> （auto memory 是 CLAUDE.md 的"Claude 写"的对应物——CLAUDE.md 是你写的。）

还有一个对所有机制都成立的共同前提，必须放在最前面理解：

> "Both are loaded at the start of every conversation. Claude treats them as context, not enforced configuration."
> （两者都在每次对话开始时加载。Claude 把它们当作**上下文**，而不是**强制执行的配置**。）

这意味着：两者都是"软性"建议——具体、简洁的指令 Claude 更容易遵循；**若要"无论 Claude 怎么想都必须拦截某动作"，官方明确指向 `PreToolUse` hook**，而不是依赖这两个机制。

## 二、CLAUDE.md：你写给人也写给 Claude 的持久指令

### 是什么

> "CLAUDE.md files are markdown files that give Claude persistent instructions for a project, your personal workflow, or your entire organization. You write these files in plain text; Claude reads them at the start of every session."
> （CLAUDE.md 是为项目、你的个人工作流、或整个组织提供持久指令的 markdown 文件。你用纯文本写，Claude 在每个会话开始时读取。）

一个常被忽略的实现细节：CLAUDE.md 是跟在系统提示之后、以**用户消息**形式送达的，而不是系统提示本身的一部分——

> "CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there's no guarantee of strict compliance."
> （CLAUDE.md 内容是系统提示之后的一条**用户消息**，不是系统提示本身的一部分。Claude 会读它、尽量遵守它，但并不保证严格服从。）

这从机制上解释了它为何只是"软性上下文"——这也正是官方排障章节的起点。

**作者只有一个：你**（或你的团队 / 组织）。Claude 只读不写。正因如此，CLAUDE.md 适合放"你希望 Claude 每个会话都握着的事实"：

- 构建 / 测试命令、代码约定、项目布局
- "总是做 X"类的硬规则
- 需要**跨会话稳定**、且**由团队共享**（可进版本控制）的信息

官方对"什么时候该往 CLAUDE.md 里加"给了四个触发器：

> "Treat CLAUDE.md as the place you write down what you'd otherwise re-explain. Add to it when: Claude makes the same mistake a second time / A code review catches something Claude should have known about this codebase / You type the same correction or clarification into chat that you typed last session / A new teammate would need the same context to be productive."
> （把 CLAUDE.md 当作"你本来会反复解释的内容"的存放处。当以下情况发生时加进去：Claude 第二次犯同一个错；code review 抓到 Claude 本应知道的代码库知识；你上次会话打过的纠正这次又打了一遍；新同事需要同样的上下文才能上手。）

### 放在哪里（作用域从大到小）

CLAUDE.md 可以放在多个位置，官方按加载顺序（范围从大到小）列出：

| 作用域 | 位置 | 用途举例 | 共享对象 |
|---|---|---|---|
| **Managed policy（托管策略）** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | 公司编码规范、安全策略、合规要求 | 组织内所有用户 |
| **User（用户）** | `~/.claude/CLAUDE.md` | 个人代码风格偏好、个人工具快捷键 | 仅你自己（所有项目） |
| **Project（项目）** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目架构、编码规范、常用工作流 | 团队成员（通过版本控制） |
| **Local（本地私有）** | `./CLAUDE.local.md`（需加进 `.gitignore`） | 你的沙箱 URL、偏好的测试数据 | 仅你自己（当前项目） |

关键点：**工作目录之上的 CLAUDE.md / CLAUDE.local.md 在启动时整文件加载**（"loaded in full at launch"）；**子目录里的 CLAUDE.md 按需加载**——只有 Claude 读取该子目录下的文件时才进入上下文。

### 怎么写（官方建议）

- **大小**：每个 CLAUDE.md 目标 **200 行以内**。CLAUDE.md 无论多长都会整文件加载（这点与 auto memory 的 200 行上限不同），越短遵循度越高。
- **结构**：用 markdown 标题和列表分组；Claude 和读者一样扫结构。
- **具体**："Use 2-space indentation" 好于 "Format code properly"；"Run `npm test` before committing" 好于 "Test your changes"。
- **一致性**：两条规则互相矛盾时，Claude 可能任选其一，要定期清理冲突指令。
- 可用 `@path/to/import` 语法导入其他文件（递归最多 4 层）；`CLAUDE.local.md` 的跨 worktree 个人偏好可导入 `~/.claude/...` 下的文件。
- `CLAUDE.md` 里的块级 HTML 注释（`<!-- ... -->`）在注入上下文前会被剥掉，可用来给人类维护者留批注而不占 token。

### 一个易混淆点：AGENTS.md

> "Claude Code reads `CLAUDE.md`, not `AGENTS.md`."
> （Claude Code 读的是 `CLAUDE.md`，不是 `AGENTS.md`。）

若仓库已为其他编程 agent 维护 `AGENTS.md`，官方建议建一个 `CLAUDE.md` 用 `@AGENTS.md` 导入（或做 symlink），让两套工具读同一份指令而不重复维护。

## 三、Auto memory（自动记忆）：Claude 写给自己看的笔记

### 是什么

> "Auto memory lets Claude accumulate knowledge across sessions without you writing anything. Claude saves notes for itself as it works: build commands, debugging insights, architecture notes, code style preferences, and workflow habits."
> （自动记忆让 Claude 在**你什么都不写**的情况下跨会话积累知识。Claude 干活时会给自己记笔记：构建命令、调试心得、架构说明、代码风格偏好、工作流习惯。）

**作者只有一个：Claude**。你完全不参与书写。它的"记忆触发"不是每次会话都存，而是 Claude 自己判断：

> "Claude doesn't save something every session. It decides what's worth remembering based on whether the information would be useful in a future conversation."
> （Claude 不是每个会话都存东西。它根据"这条信息对未来的对话是否有用"来判断什么值得记。）

界面里的体现：当你看到 Claude Code 提示 **"Saved 2 memories" / "Recalled 2 memories"** 时，就是 Claude 在写入 / 读取 `~/.claude/projects/<project>/memory/`。

### 开关与配置

| 方式 | 做法 |
|---|---|
| **默认** | 开（"Auto memory is on by default."） |
| **会话内切换** | `/memory` 命令里的 auto memory toggle |
| **用户设置关** | `~/.claude/settings.json` 中 `"autoMemoryEnabled": false` |
| **单项目关** | 该项目 `.claude/settings.json` 中 `"autoMemoryEnabled": false` |
| **环境变量全局关** | `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` |
| **自定义存储目录** | `settings.json` 中 `"autoMemoryDirectory": "~/my-custom-memory-dir"`（须绝对路径或以 `~/` 开头） |

### 存在哪里

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 索引：每个会话加载前 200 行或 25KB
├── debugging.md       # 主题文件：按需读取
├── api-conventions.md # ...
└── ...
```

- `<project>` 由 **git 仓库路径**推导，所以**同一仓库的所有 worktree 和子目录共享同一个 auto memory 目录**；不在 git 仓库内则用项目根目录。
- **Auto memory 是机器本地的**（"Auto memory is machine-local"），不跨机器、不跨云端环境共享。
- `MEMORY.md` 是入口索引。**每次会话开始加载前 200 行或前 25KB（先到者为准）**；超过该阈值的内容不会被加载。
- Claude 会把过长的 `MEMORY.md` 保持精简：把细节移入独立主题文件。接近上限时 Claude Code 会提醒 Claude 缩短它；超过上限则写入仍成功，但 Claude Code 会返回错误让 Claude 重写索引。
- **主题文件（如 `debugging.md`）不随会话启动加载**，Claude 在需要时用标准文件工具**按需读取**（"Topic files like `debugging.md` or `patterns.md` are not loaded at startup. Claude reads them on demand"）。

> 对比要点：这个 200 行 / 25KB 的上限**只针对 `MEMORY.md` 索引**；CLAUDE.md 是无论多长都整文件加载（只是长了对遵循度不利）。

### 可审计、可编辑

> "Auto memory files are plain markdown you can edit or delete at any time."
> （auto memory 文件是纯 markdown，你随时可以编辑或删除。）

用 `/memory` 可浏览、打开、编辑记忆文件；用 `/context` 可确认当前会话实际加载了哪些 memory 文件。官方特意点明：你虽然可以编辑或删除 `MEMORY.md`，但 **"Claude will keep updating it"**（Claude 会继续更新它）。

### subagent 也有自己的 auto memory

> "Subagents can also maintain their own auto memory."
> （subagent 也可以维护自己的 auto memory。）

subagent 的持久记忆独立于主会话：在 subagent frontmatter 设 `memory: project`（写入 `.claude/agent-memory/`，团队共享）、`memory: local`（`.claude/agent-memory-local/`，不进版本控制）、或 `memory: user`（`~/.claude/agent-memory/`，跨项目）。每个 subagent 读写自己的 `MEMORY.md`，**不读你主会话的**；主会话的 auto memory 也不会加载进普通 subagent（fork 除外）。

## 四、一张表看清"区分"本身

把两者并排看，最容易踩混的维度如下：

| 区分维度 | CLAUDE.md | Auto memory |
|---|---|---|
| **谁写** | 你 / 团队 / 组织（人） | Claude（模型） |
| **内容性质** | 指令与规则（你想要它怎么做） | 经验与模式（它发现该怎么做） |
| **是否进版本控制** | 通常要提交（项目级，团队共享） | **不**，机器本地（`~/.claude/projects/`） |
| **作用域** | 项目 / 用户 / 组织 / 本地私有（四种位置） | 按 git 仓库，跨 worktree 共享 |
| **加载时机** | 启动时整文件加载；子目录的按需加载 | 启动时只加载 `MEMORY.md` 前 200 行 / 25KB；主题文件按需读取 |
| **大小限制** | 无硬性上限（建议 <200 行） | `MEMORY.md` 硬上限：200 行 / 25KB |
| **会话中编辑是否立即生效** | **不生效**（根级与用户级读一次后常驻内存） | 即写即读（Claude 会话内持续读写） |
| **主动触发方式** | 对 Claude 说"add this to CLAUDE.md"，或自己 `/memory` 编辑 | 对 Claude 说"remember that…"，或它自行判断 |
| **强制力** | 软性（context，非强制配置） | 软性（context，非强制配置） |

## 五、什么时候该用哪一个（官方决策指引）

官方在 `memory.md` 里给了一个非常直接的分流规则，值得原文引用：

> "When you ask Claude to remember something, like 'always use pnpm, not npm' or 'remember that the API tests require a local Redis instance,' Claude saves it to auto memory. To add instructions to CLAUDE.md instead, ask Claude directly, like 'add this to CLAUDE.md,' or edit the file yourself via /memory."
> （当你叫 Claude "记住"某件事，比如"总是用 pnpm，不要用 npm"或"记住 API 测试需要本地 Redis 实例"，Claude 会把它存进 **auto memory**。想加进 **CLAUDE.md** 的话，就直接叫 Claude"把这个加到 CLAUDE.md"，或用 `/memory` 自己编辑。）

据此可以总结成一张"翻译成具体动作"的决策表：

| 场景 | 该用哪个 | 理由 |
|---|---|---|
| "Remember that…"——你想让 Claude 自己记住 | **Auto memory** | 它按"对未来是否有用"判断，零维护 |
| 你想给 Claude 一条**持久规则 / 规范** | **CLAUDE.md** | 你写、你掌控、可进版本控制、团队共享 |
| 同一条纠正你重复打了第二遍 | **CLAUDE.md** | 官方四触发器之一："same correction… you typed last session" |
| 想让**组织全员**都遵守某规范 | **CLAUDE.md（managed policy 或项目级）** | 作用域覆盖组织 / 项目 |
| 想让 Claude 跨会话记得**构建命令、调试套路** | **Auto memory** 优先，也可写进 CLAUDE.md | 官方归为 auto memory 的典型内容 |
| 要**硬性拦截**某动作（不管 Claude 怎么想） | **都不是——用 PreToolUse hook** | "Claude treats them as context, not enforced configuration" |

### `/compact`（压缩上下文）后谁存活

官方 `context-window` 页给了一张"压缩后存活"表，是另一处能一眼看出二者差异的地方：

| 机制 | 压缩（`/compact`）之后 |
|---|---|
| 项目根 CLAUDE.md 与无 `paths` 的 rules | **从磁盘重新注入**（Re-injected from disk） |
| **Auto memory** | **从磁盘重新注入**（Re-injected from disk） |
| 带 `paths:` 的 rules | 丢失，直到再次读到匹配文件（Lost until a matching file is read again） |
| 子目录里的嵌套 CLAUDE.md | 丢失，直到再次读到该子目录文件（Lost until a file in that subdirectory is read again） |
| 已调用的 skill 正文 | 重新注入（每 skill 上限 5000 token、总计 25000 token，最旧的先丢） |
| Hooks | 不适用（hook 是代码执行，不是上下文） |

要点：**项目根 CLAUDE.md 与 auto memory 都能扛过 `/compact`**（压缩后从磁盘重读、重新注入）；**嵌套子目录 CLAUDE.md 与 `paths:` 规则不能**——它们加载进消息历史后随压缩一起被摘要掉，要等下次读到匹配文件才重新加载。想让某条规则扛过压缩，就去掉 `paths:` 或挪进项目根 CLAUDE.md。

几个反复出现的易混淆点：

- **"会话中编辑 CLAUDE.md 不生效"是 prompt caching 的连带行为。** 根级与用户级 CLAUDE.md 在会话开始时读一次并常驻内存（官方原话："read once at session start and held in memory"），中途编辑既不清缓存、也不生效，新内容要等 `/clear`、`/compact` 或重启才加载。**例外**：尚未加载过的子目录 CLAUDE.md 与带 `paths:` 的规则，在首次加载前编辑是生效的。
- **auto memory 的写入本身不占你的版本控制**，但它的主题文件一样是 markdown，随时可读可改——"Claude 写的"不等于"不可见、不可控"。
- **两者都会在启动时占用上下文 token**。`prompt-caching` 文档把 `CLAUDE.md, auto memory, unscoped rules` 同列为"Project context"层，随会话开始 / `/clear` / `/compact` 而变。省上下文的思路对两者一致：CLAUDE.md 靠拆分 / 子目录按需加载，auto memory 靠 Claude 把细节挪进主题文件。

## 六、2026-08 核验时官方文档的新增细节

> 以下细节是 2026-08-04 重抓官方 `memory` 页确认的较新补充，逐字核对。前几节成稿时未含，单独列出以免漏掉；多为版本演进新增（文内标注版本号）。

### 1. `MEMORY.md` 上限的计量方式：先剥离再计数（v2.1.211+）

auto memory 的 200 行 / 25KB 上限，计量的是"实际加载进上下文的内容"，不是原始文件：

> "The check measures only the content that loads: YAML frontmatter and block-level HTML comments are stripped before the index is loaded, so they don't count toward the limits. Before v2.1.211, Claude Code measured the raw file, and frontmatter or comments could trigger the error even when the loaded content fit."
> （上限只计量**加载时进入上下文的内容**：YAML frontmatter 和块级 HTML 注释在索引加载前先被剥离、不计入限制。v2.1.211 之前按原始文件计量，frontmatter 或注释可能在内容其实没超限时误触发报错。）

### 2. auto memory 文件的 `modified` 时间戳（v2.1.214+）

> "When Claude writes a memory file that begins with YAML frontmatter, Claude Code records the write time in a `modified` frontmatter field as an ISO 8601 timestamp."
> （Claude 写入以 YAML frontmatter 开头的记忆文件时，Claude Code 会在 frontmatter 里用 ISO 8601 时间戳记录写入时间。）

用途是让记忆的**时效性可查**——你（以及 Claude 读回时）都能看到这条事实是什么时候写的。Claude Code 不会给原本没有 frontmatter 的文件补 frontmatter；任何带 frontmatter 的文件，下次被写入时都会得到该字段。

### 3. 托管 CLAUDE.md 可直接写进 `managed-settings.json` 的 `claudeMd` 键

组织级指令不一定要单独部署一个 CLAUDE.md 文件：

> "The `claudeMd` key lets you put managed CLAUDE.md content directly inside `managed-settings.json` instead of deploying a separate file."
> （`claudeMd` 键让你把托管 CLAUDE.md 的内容直接写进 `managed-settings.json`，不用单独部署文件。）

官方顺带点明了与 managed settings 的分工，这是"上下文 vs 强制"原则的另一种表述：

> "A managed CLAUDE.md and managed settings serve different purposes. Use settings for technical enforcement and CLAUDE.md for behavioral guidance. … Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer."
> （托管 CLAUDE.md 与 managed settings 分工不同：**settings 管技术上的强制**（deny 命令、沙箱、env、登录方式），**CLAUDE.md 管行为引导**（代码风格、合规提醒、行为指令）。settings 规则由客户端强制执行、与 Claude 是否愿意无关；CLAUDE.md 只是塑造行为，不是硬性执行层。）

### 4. `claudeMdExcludes`：monorepo 里排除其他团队的 CLAUDE.md

> "In large monorepos, ancestor CLAUDE.md files may contain instructions that aren't relevant to your work. The `claudeMdExcludes` setting lets you skip specific files by path or glob pattern."
> （大 monorepo 里，祖先目录的 CLAUDE.md 可能含与你工作无关的指令。`claudeMdExcludes` 让你按路径或 glob 跳过指定文件。）

建议配在 `.claude/settings.local.json`（保持机器本地）。**托管策略 CLAUDE.md 不可被排除**（"Managed policy CLAUDE.md files cannot be excluded"），组织级指令始终生效。

### 5. `/doctor` 给 CLAUDE.md 提删减建议（v2.1.206+）

> "The `/doctor` checkup proposes trims for a checked-in CLAUDE.md: it cuts content Claude can derive from the codebase, such as directory layouts, dependency lists, and architecture overviews, and keeps pitfalls, rationale, and conventions that differ from tool defaults."
> （`/doctor` 会为已提交的 CLAUDE.md 提出删减建议：**砍掉** Claude 能从代码库推导的内容——目录布局、依赖清单、架构总览；**保留**坑、理由、以及偏离工具默认的约定。）

这正好落地了第七节"过度指定 CLAUDE.md"的自检法，可定期跑一遍。

### 6. `InstructionsLoaded` hook：排查"到底哪些指令文件被加载"

> "Use the `InstructionsLoaded` hook to log exactly which instruction files are loaded, when they load, and why."
> （用 `InstructionsLoaded` hook 记录到底是哪些指令文件被加载、何时加载、为什么加载。）

配合 `/context`（查当前会话实际加载清单）使用，是排查 path-scoped rule 或子目录懒加载 CLAUDE.md 未生效的首选手段。

### 7. `--add-dir` 额外目录的记忆文件默认不加载（`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` 才加载）

`--add-dir` 给出的额外目录，其 CLAUDE.md 默认不加载；设环境变量 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` 后，才会加载其中的 `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md`、`CLAUDE.local.md`。

### 8. 交互式 `/init`（`CLAUDE_CODE_NEW_INIT=1`）

> "Set `CLAUDE_CODE_NEW_INIT=1` to enable an interactive multi-phase flow. `/init` asks which artifacts to set up: CLAUDE.md files, skills, and hooks."
> （设 `CLAUDE_CODE_NEW_INIT=1` 启用交互式多阶段流程：`/init` 会问你想要哪些产物——CLAUDE.md、skills、hooks，然后派 subagent 探索代码库、补充问题、写出可审阅的提案。）

### 9. 外部导入的审批弹窗（安全）

项目级记忆文件里的导入，若路径解析到**工作目录之外**（如 `@~/.claude/my-project-instructions.md`），首次遇到会弹审批框列出文件；用户拒绝后该导入保持禁用、不再弹窗。用户级（`~/.claude/CLAUDE.md`、`~/.claude/rules/`）的导入是你自己写的，直接加载、不弹窗。这是防"别人提交的共享项目导入坑你"的保护。

### 10. 想放系统提示层：`--append-system-prompt`

> "For instructions you want at the system prompt level, use `--append-system-prompt`. This must be passed every invocation, so it's better suited to scripts and automation than interactive use."
> （想在系统提示层放指令，用 `--append-system-prompt`。它每次调用都要传，更适合脚本/自动化而非交互使用。）

## 七、相邻机制别放错：CLAUDE.md / rules / skills / hooks

官方多次强调"选错机制"是常见问题。把 memory、best-practices 的信息归纳成一张决策表，避免把"该放 CLAUDE.md 的东西"误放进别的机制、或反过来：

| 机制 | 谁写 | 何时加载 | 适合放什么 | 不适合放什么 |
|---|---|---|---|---|
| **CLAUDE.md** | 人 | 每次会话（全量） | 适用于整个项目/会话的约定：构建命令、编码规范、架构决策、"always do X" | 一次性流程、只影响某目录的规则、Claude 读代码就能推断出来的东西 |
| **Auto memory** | Claude | 每次会话（前 200 行/25KB 的 MEMORY.md） | Claude 发现的经验：构建命令、调试套路、你的偏好 | 需要你主动控制的指令——它随时可能被改写/精简 |
| **`.claude/rules/`**（含路径作用域） | 人 | 每次会话，或只在匹配 `paths` 的文件被读取时 | 按目录/文件类型划分的约定（如 `src/api/**/*.ts` 专属规则） | 需要全局可见的内容 |
| **Skills** | 人 | **按需**：被调用时或 Claude 判断相关时 | 多步骤流程、只在特定场景用到的领域知识 | 每条会话都该在上下文里的常驻规则 |
| **Hooks** | 人（配置） | 固定生命周期事件（确定性执行） | 必须无条件执行的动作（如提交前必跑 lint） | 建议性、可协商的行为 |

关键判断词来自官方：

> "If an entry is a multi-step procedure or only matters for one part of the codebase, move it to a skill or a path-scoped rule instead."
> （如果一条内容是**多步骤流程**或**只影响代码库一部分**，移去 skill 或 path-scoped rule，而不是留在 CLAUDE.md 里。）

另外注意一个来自 `.claude` 目录文档的对照：**settings.json 是 enforced（强制执行），CLAUDE.md 是 guidance（指导）**——

> "Unlike CLAUDE.md, which Claude reads as guidance, these are enforced whether Claude follows them or not."
> （settings.json 与 CLAUDE.md 不同——CLAUDE.md 是 Claude 当作**指导**来读的，而 settings 是**不管 Claude 是否遵守都会被强制执行**的。）

这正好呼应第一节"两者都只是上下文、不是强制配置"：**想让某行为"一定会发生"，写 settings / hook，别指望记忆文件。**

最后是 CLAUDE.md 自身的高频失败模式（best-practices 反复点名）：

> "**The over-specified CLAUDE.md.** If your CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise."
> （**过度指定的 CLAUDE.md。** 文件太长时，Claude 会无视其中一半，因为重要规则淹没在噪音里。）

官方给的行级自检法："For each line, ask: *'Would removing this cause Claude to make mistakes?'* If not, cut it."（对每一行问"删掉这行 Claude 会犯错吗？"不会就删掉。）

## 八、一句话收尾

> 作者决定一切：**CLAUDE.md 是"你写、Claude 读"的持久指令，auto memory 是"Claude 写、Claude 读"的经验笔记**。前者你完全掌控、可版本化、适合团队共享的规范；后者零维护、自动积累、机器本地、适合"Claude 从纠正中自学习"。两者都只是上下文而非强制配置——需要硬约束时，用 hook 而非记忆文件。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 获取；第六节新增细节为重抓 `memory` 页时逐字核对）：

- **How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （本文主体：CLAUDE.md vs auto memory 官方对比表、CLAUDE.md 四种作用域与写法建议、auto memory 开关 / 存储位置 / MEMORY.md 索引与 200 行 / 25KB 上限、`/memory` 命令、故障排查、"remember that…" vs "add this to CLAUDE.md" 的分流原话；第六节新增细节——MEMORY.md 先剥离 frontmatter/注释再计数、`modified` 时间戳、`claudeMd` 键、`claudeMdExcludes`、`/doctor`、`InstructionsLoaded`、`--add-dir`、`CLAUDE_CODE_NEW_INIT`、外部导入审批、`--append-system-prompt`）
- **Explore the .claude directory** — https://code.claude.com/docs/en/claude-directory
  （.claude 目录各文件定位：CLAUDE.md"committed / 项目指令"、`projects/<project>/memory/`"auto memory：Claude 写给自己跨会话的笔记"、MEMORY.md 与主题文件的生成方式、"settings 是强制、CLAUDE.md 是 guidance" 的对比）
- **How Claude Code uses prompt caching** — https://code.claude.com/docs/en/prompt-caching
  （Project context 层 = CLAUDE.md + auto memory + unscoped rules；"编辑根级/用户级 CLAUDE.md 会话内不生效、等 `/clear`/`/compact`/重启"的原话；子目录 CLAUDE.md 与 `paths:` 规则按需加载的例外）
- **Glossary** — https://code.claude.com/docs/en/glossary
  （"Auto memory is the Claude-written counterpart to CLAUDE.md, which you write" 原话出处）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （启动加载项清单：auto memory（MEMORY.md）、`~/.claude/CLAUDE.md`、项目 CLAUDE.md 都进上下文；"What survives compaction" 存活表）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"over-specified CLAUDE.md" 失败模式与"Would removing this cause Claude to make mistakes?"行级自检法、CLAUDE.md 该放/不该放内容清单、与 skills 的取舍）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md 管理的实操）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（agentic search 与上下文管理的官方立场）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
