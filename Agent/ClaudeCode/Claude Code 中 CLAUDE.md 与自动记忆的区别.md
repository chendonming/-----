# Claude Code 中 CLAUDE.md 与自动记忆（auto memory）的区别

> **一句话总结**：官方把 Claude Code 的记忆系统定义为**两套互补机制**——**CLAUDE.md 是你（或团队）手写给 Claude 的持久指令，auto memory 是 Claude 根据你的纠正和偏好自己记的笔记**。两者的根本分工是：**想主动引导 Claude 的行为 → 写 CLAUDE.md；想让它从你的纠正中自动学习、无需手动维护 → 开 auto memory（默认开）**。它们每次会话都会加载，但本质都是"上下文"而非"强制配置"；此外还有 skills（按需加载的任务流程）、`.claude/rules/`（路径作用域规则）、hooks（确定性执行）等相邻机制，放错地方会互相干扰。
>
> 本文基于 Claude Code 官方文档 `How Claude remembers your project`（即 memory）、`Best practices for Claude Code` 整理，文末附参考来源。

很多人第一次接触 Claude Code 时，会困惑于一个看起来矛盾的现象：官方既说"每次会话都从空白上下文开始"，又说"Claude 记得你的项目"。答案在于：**跨会话的记忆不是单一系统，而是两个**——一个由人写，一个由机器写。

---

## 一、官方对两套记忆系统的总定义

官方文档 `How Claude remembers your project` 开篇就点明了这两套机制的分工：

> "Give Claude persistent instructions with CLAUDE.md files, and let Claude accumulate learnings automatically with auto memory."
> （用 CLAUDE.md 文件给 Claude 持久指令，并让 Claude 通过 auto memory 自动积累经验。）

以及：

> "Each Claude Code session begins with a fresh context window. Two mechanisms carry knowledge across sessions:
> * **CLAUDE.md files**: instructions you write to give Claude persistent context
> * **Auto memory**: notes Claude writes itself based on your corrections and preferences"
> （每次 Claude Code 会话都从全新的上下文窗口开始。有两个机制把知识带跨会话：**CLAUDE.md 文件**——你写下的、给 Claude 的持久上下文；**auto memory**——Claude 根据你的纠正和偏好自己写的笔记。）

注意"fresh context window"这个前提：记忆系统存在的唯一理由，就是**抵消"每次会话从零开始"这件事**。

## 二、CLAUDE.md：你写给 Claude 的持久指令

CLAUDE.md 是普通 markdown 文件，由**人**编写。官方定义：

> "CLAUDE.md is a special file that Claude reads at the start of every conversation. Include Bash commands, code style, and workflow rules. This gives Claude persistent context it can't infer from code alone."
> （CLAUDE.md 是 Claude 在每次对话开始时读取的特殊文件。放入 Bash 命令、代码风格和工作流规则，它给 Claude 提供**光靠读代码推断不出来**的持久上下文。）

关键属性：

- **由你写**：写在版本控制里，随仓库共享给团队。
- **每次会话全量加载**：`CLAUDE.md` 文件无论多长都完整加载（官方明确 "CLAUDE.md files are loaded in full regardless of length"），但越长服从度越差。
- **多级作用域**：官方给出的加载顺序（从最宽到最具体）：

| 作用域 | 位置 | 用途示例 |
|---|---|---|
| 组织托管策略 | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`，Linux `/etc/claude-code/CLAUDE.md`，Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | 公司编码规范、安全策略，用户无法排除 |
| 用户级 | `~/.claude/CLAUDE.md` | 个人在所有项目的偏好 |
| 项目级 | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享的项目架构、规范，走版本控制 |
| 本地级 | `./CLAUDE.local.md`（建议加入 `.gitignore`） | 个人私有的项目偏好 |

- **该写什么**：官方给了明确的判断标准（"When to add to CLAUDE.md"）——当出现这些信号时写进去：
  - Claude 第二次犯同样的错误；
  - 一次 code review 抓到了 Claude 本该知道的代码库事实；
  - 你把上一轮会话里打过的同样的纠正，又打了一遍；
  - 一个新同事需要同样的上下文才能高效干活。
  - 一句话："Treat CLAUDE.md as the place you write down what you'd otherwise re-explain."（把 CLAUDE.md 当成"你本来会反复解释的东西"的存放处。）

## 三、Auto memory：Claude 自己记的笔记

Auto memory 是官方相对较新的能力，它的价值在于**零维护**。官方定义：

> "Auto memory lets Claude accumulate knowledge across sessions without you writing anything. Claude saves notes for itself as it works: build commands, debugging insights, architecture notes, code style preferences, and workflow habits."
> （Auto memory 让你**什么都不用写**，Claude 就能跨会话积累知识。它在工作时为自己记笔记：构建命令、调试洞察、架构笔记、代码风格偏好、工作流习惯。）

关键属性：

- **默认开启**，可关：`/memory` 里开关（写入用户级 `autoMemoryEnabled` 设置）、项目级 `settings.json` 里 `"autoMemoryEnabled": false`、或环境变量 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。
- **不是每次都记**：官方明确说——"Claude doesn't save something every session. It decides what's worth remembering based on whether the information would be useful in a future conversation."（Claude 不是每次会话都记东西，它判断**"这条信息未来对话是否用得上"**来决定是否值得记。）
- **存储位置**：`~/.claude/projects/<project>/memory/`，每个 git 仓库一个目录，**同仓库的所有 worktree 和子目录共享**。
- **目录结构**：一个 `MEMORY.md` 入口文件 + 按主题拆分的可选文件（如 `debugging.md`、`api-conventions.md`）。`MEMORY.md` 是索引，Claude 全程读写它来跟踪"什么存在哪"。
- **加载上限**：只有 `MEMORY.md` 的前 **200 行或前 25KB**（先到者）在每次会话启动时加载；超出部分不加载。细节笔记拆到主题文件里，按需用文件工具读取。这与 CLAUDE.md 的"全量加载"形成鲜明对比。
- **机器本地**："Auto memory is machine-local. … Files are not shared across machines or cloud environments."（不跨机器/云共享。）
- **界面信号**：当你看到 Claude Code 界面里 "Saved 2 memories" / "Recalled 2 memories" 这类消息，就是 Claude 正在读写 `~/.claude/projects/<project>/memory/`。

## 四、关键差异：一张表讲清

官方在 memory 文档里直接给了对比表，逐字照录并译：

| | **CLAUDE.md files** | **Auto memory** |
|---|---|---|
| **Who writes it（谁写）** | You（你） | Claude |
| **What it contains（内容）** | Instructions and rules（指令与规则） | Learnings and patterns（经验与模式） |
| **Scope（作用域）** | Project, user, or org（项目/用户/组织） | Per repository, shared across worktrees（每个仓库，跨 worktree 共享） |
| **Loaded into（加载进）** | Every session（每次会话） | Every session — first 200 lines or 25KB（每次会话——前 200 行或 25KB） |
| **Use for（用于）** | Coding standards, workflows, project architecture（编码规范、工作流、项目架构） | Build commands, debugging insights, preferences Claude discovers（构建命令、调试洞察、Claude 发现的偏好） |

官方给的选型口诀就一句：

> "Use CLAUDE.md files when you want to guide Claude's behavior. Auto memory lets Claude learn from your corrections without manual effort."
> （**想引导 Claude 的行为 → 用 CLAUDE.md；想让 Claude 从你的纠正中学习、免去手动维护 → 用 auto memory。**）

还有一个容易被忽略的共同点——两者都**不是强制配置**：

> "Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead."
> （Claude 把两者都当作上下文，而非强制配置。**无论如何都要阻止某个动作时，改用 PreToolUse hook。**）

也就是说：CLAUDE.md 和 auto memory 都只是"喂给模型的上下文"，Claude 会读并尽量遵守，但不保证 100% 执行；要"硬性拦截"得用 hook。

## 五、内容放哪：CLAUDE.md vs auto memory vs skills vs rules vs hooks

官方多次强调"选错机制"是常见问题。把 memory、best-practices、features-overview 的信息归纳成一张决策表：

| 机制 | 谁写 | 何时加载 | 适合放什么 | 不适合放什么 |
|---|---|---|---|---|
| **CLAUDE.md** | 人 | 每次会话（全量） | 适用于整个项目/会话的约定：构建命令、编码规范、架构决策、"always do X" | 一次性流程、只影响某目录的规则、Claude 读代码就能推断出来的东西 |
| **Auto memory** | Claude | 每次会话（前 200 行/25KB 的 MEMORY.md） | Claude 发现的经验：构建命令、调试套路、你的偏好 | 需要你主动控制的指令——它随时可能被改写/精简 |
| **`.claude/rules/`**（含路径作用域） | 人 | 每次会话，或只在匹配 `paths` 的文件被读取时 | 按目录/文件类型划分的约定（如 `src/api/**/*.ts` 专属规则） | 需要全局可见的内容 |
| **Skills** | 人 | **按需**：被调用时或 Claude 判断相关时 | 多步骤流程、只在特定场景用到的领域知识 | 每条会话都该在上下文里的常驻规则 |
| **Hooks** | 人（配置） | 固定生命周期事件（确定性执行） | 必须无条件执行的动作（如提交前必跑 lint） | 建议性、可协商的行为 |

关键判断词来自官方：

- **"Keep it to facts Claude should hold in every session: build commands, conventions, project layout, 'always do X' rules. If an entry is a multi-step procedure or only matters for one part of the codebase, move it to a skill or a path-scoped rule instead."**（CLAUDE.md 只放"每条会话都该持有的事实"；多步骤流程或只影响代码库一部分的内容，移去 skill 或 path-scoped rule。）
- **"CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand without bloating every conversation."**（CLAUDE.md 每次会话都加载，只放普适内容；偶尔才相关的内容用 skill，按需加载、不撑爆每次对话。）

## 六、实战建议与常见坑

### 1. CLAUDE.md 宁短勿长，否则 Claude 会"无视"它

这是官方反复点名的高频失败模式：

> "**The over-specified CLAUDE.md.** If your CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise."
> （**过度指定的 CLAUDE.md。** 文件太长时，Claude 会无视其中一半，因为重要规则淹没在噪音里。）

官方给出的行级自检法：

> "Keep it concise. For each line, ask: *'Would removing this cause Claude to make mistakes?'* If not, cut it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"
> （保持精简。对每一行问："删掉这行 Claude 会犯错吗？"不会就删掉。臃肿的 CLAUDE.md 会让 Claude 忽略你真正的指令！）

方向性建议：**"Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts."**（像对待代码一样对待 CLAUDE.md：出错时审查、定期精简、通过观察 Claude 行为是否真的变化来验证改动。）

### 2. 用 `/init` 起步，别从空白开始

> "Run `/init` to generate a starter CLAUDE.md file based on your current project structure, then refine over time."
> （跑 `/init` 基于当前项目结构生成一份起步 CLAUDE.md，之后逐步打磨。）

官方补充：`/init` 分析代码库识别构建系统、测试框架、代码模式；已有 CLAUDE.md 时 `/init` 只给改进建议、不覆盖。

### 3. CLAUDE.md 里该放/不该放的内容清单

官方 best-practices 给的对照表，值得逐条过：

| ✅ 放进去 | ❌ 别放 |
|---|---|
| Claude 猜不出来的 Bash 命令 | Claude 读代码就能推断出来的东西 |
| 与默认不同的代码风格规则 | 标准语言惯例（Claude 本来就懂） |
| 测试指令、偏好的测试运行器 | 详细 API 文档（给链接即可） |
| 仓库礼仪（分支命名、PR 约定） | 频繁变化的信息 |
| 项目专属的架构决策 | 长篇解释或教程 |
| 开发环境怪癖（必需的 env var） | 代码库的逐文件描述 |
| 常见坑、非显而易见的坑点 | 自明式实践（如"写整洁代码"） |

### 4. Auto memory 的触发与"升级"路径

- 你说"**always use pnpm, not npm**"、"**remember that the API tests require a local Redis instance**"这类话时，Claude 会存进 auto memory。
- 想改成 CLAUDE.md，则明说"**add this to CLAUDE.md**"，或自己用 `/memory` 打开编辑。
- Auto memory 是**可审阅、可编辑**的普通 markdown：`/memory` 可以浏览/打开/删除，`/context` 可以确认当前会话实际加载了哪些记忆文件。

### 5. 谁的记忆在 `/compact`（压缩上下文）后存活？

官方明确：**项目根目录的 CLAUDE.md 存活**——压缩后 Claude 会从磁盘重读并重新注入；子目录里的嵌套 CLAUDE.md 不会自动重新注入，要等下次读到该子目录文件。会话里口头给过、但没写进任何文件的指令，压缩后丢失。这正说明：**想让一句话持久，就得落盘到 CLAUDE.md**。

### 6. 一个易混点：auto memory 是"每仓库"的，CLAUDE.md 可以是"全局"的

Auto memory 天然按 git 仓库隔离（跨 worktree 共享），而 CLAUDE.md 有用户级（`~/.claude/CLAUDE.md`）和项目级之分。所以**跨项目的个人偏好靠用户级 CLAUDE.md，不靠 auto memory**。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 通过 web_fetch 获取）：

- **How Claude remembers your project（memory）** — https://code.claude.com/docs/en/memory
  （核心页：CLAUDE.md vs auto memory 对比表、CLAUDE.md 各作用域与加载规则、auto memory 的开关/存储/`MEMORY.md` 200 行 25KB 上限、`/memory` 与 `/context` 操作、故障排查）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"Write an effective CLAUDE.md"：`/init`、行级自检法、CLAUDE.md 该放/不该放内容表、"over-specified CLAUDE.md" 失败模式、与 skills 的取舍）

> 相关文档：`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（分层 CLAUDE.md 在大仓库中的用法）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md 管理）、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`。
