# Claude Code 最佳实践：提示词与 CLAUDE.md 的官方建议

> **一句话总结**：Claude Code 官方关于"怎么用好它"的建议可以浓缩成两件事——**把提示词写具体、给 Claude 一个能验证结果的检查**；以及**用 CLAUDE.md 把跨会话的常识固化下来**（常驻但精简，臃肿会反噬）。几乎所有官方最佳实践都源于同一条约束：**上下文窗口是稀缺资源，性能随它填满而下降**。
>
> 本文全部以 Anthropic 官方文档为准，每条"官方原话"均给出英文原文与出处，文末附参考来源（含 URL 与获取日期），可逐条核对。

很多人第一次用 Claude Code，还带着和 ChatGPT 对话的习惯：一句话需求、等回复。但官方文档开篇就划清了边界：

> "Claude Code is an agentic coding environment. Unlike a chatbot that answers questions and waits, Claude Code can read your files, run commands, make changes, and autonomously work through problems while you watch, redirect, or step away entirely."
> （Claude Code 是一个**自主式（agentic）编程环境**。它不像聊天机器人那样答完就等，而是可以读你的文件、跑命令、做修改，在你观察、纠正或干脆离开的情况下**自主**把问题解决。）

这改变了工作方式：不再是"你写代码、让 Claude 审查"，而是"你描述想要什么，Claude 负责探索、规划、实现"。

而理解下面这条总纲，是写出好提示词、写好 CLAUDE.md 的共同起点。

---

## 一、总纲：上下文窗口是 Claude Code 最需要管理的资源

官方文档原话：

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."
> （大多数最佳实践都源于同一条约束：**Claude 的上下文窗口很快就会被占满，性能会随着它被填满而下降**。）

> "The context window is the most important resource to manage."
> （上下文窗口是**最需要管理**的资源。）

上下文里装着你的整个对话：每条消息、Claude 读过的每个文件、每次命令的输出。一次调试或代码库探索就可能产生几万 token；当上下文接近满载时，Claude 会"忘记"早先的指令或犯更多错。

由此可以推出官方几乎所有建议：给更精准的上下文（提示词）、把重复常识固化到按需加载的文件（CLAUDE.md / skills）、及时清理会话（`/clear`）。下文提示词与 CLAUDE.md 两部分，都是这条总纲的推论。

---

## 二、提示词：官方怎么建议你"说话"

### 1. 给 Claude 一个能验证结果的方式——最重要的建议

> **Tip（官方原文）**："Give Claude a check it can run: tests, a build, a screenshot to compare. It's the difference between a session you watch and one you walk away from."
> （给 Claude 一个它可以运行的检查：测试、构建、或一张可对比的截图。这是"你在旁观看的会话"与"你可以放手离开的会话"之间的区别。）

Claude 在"看起来做完"时就会停下。不给它可运行的检查，"看起来做完"就是它唯一的信号，你就成了验证循环——每个错误都等你来发现。检查可以是测试套件、构建退出码、linter、对输出做 diff 的脚本，或与设计稿对比的截图。

官方给出三个典型改进：

| 策略 | ❌ 之前 | ✅ 之后 |
|---|---|---|
| **提供验证标准** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing" |
| **可视化验证 UI** | "make the dashboard look better" | "\[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| **处理根因而非症状** | "the build is failing" | "the build fails with this error: \[paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

配套原则：**让 Claude 展示证据，而不是口头声称成功**——

> "Have Claude show evidence rather than asserting success: the test output, the command it ran and what it returned, or a screenshot of the result."
> （让 Claude 展示证据而不是声称成功：测试输出、它跑过的命令及返回结果、或结果截图。）

### 2. 先探索、再计划、后编码

> **Tip（官方原文）**："Separate research and planning from implementation to avoid solving the wrong problem."
> （把调研、规划与实现分开，避免解决错误的问题。）

让 Claude 直接跳去写代码，可能产出"解决了错误问题"的代码。官方推荐用 **plan mode（计划模式）** 分离探索与执行，四个阶段：Explore（`Shift+Tab` 进入，只读不改）→ Plan（产出详细计划，`Ctrl+G` 可编辑）→ Implement（退出计划模式按计划实现）→ Commit（描述性提交并开 PR）。

同时官方也提醒计划模式有额外开销：

> "If you could describe the diff in one sentence, skip the plan."
> （如果你能一句话说清这个 diff，就跳过计划。）

修 typo、加一行日志、重命名变量这类范围清楚的小改动，直接做。

### 3. 在提示词里提供具体上下文

> **Tip（官方原文）**："The more precise your instructions, the fewer corrections you'll need."
> （指令越精确，需要的纠正就越少。）

> "Claude can infer intent, but it can't read your mind."
> （Claude 能推断意图，但它不能读心。）

官方给出的四个策略：

| 策略 | ❌ 之前 | ✅ 之后 |
|---|---|---|
| **划定任务范围** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **指向信息源** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **参考现有模式** | "add a calendar widget" | "look at how existing widgets are implemented on the home page... HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget..." |
| **描述症状** | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

模糊提示词也有用武之地：正在探索、能承受纠偏成本时，一句 "what would you improve in this file?" 能抛出你没想到过的东西。

**提供富内容**（官方 Tip："Use `@` to reference files, paste screenshots/images, or pipe data directly."）：

- 用 `@` 引用文件，而不是描述代码在哪——Claude 会在响应前先读它；
- 直接粘贴 / 拖入图片；
- 给文档与 API 参考的 URL（常用域名可用 `/permissions` 加白）；
- `cat error.log | claude` 把数据直接管道传入；
- 或让 Claude 自己取：用 Bash 命令、MCP 工具、读文件获取所需上下文。

### 4. 沟通与迭代

- **像问资深工程师一样提问**（官方 Tip）："Ask Claude questions you'd ask a senior engineer." 问"日志是怎么工作的""怎么新建一个 API 端点""`foo.rs` 134 行那个 `async move {}` 是干什么的"。**不需要特殊提示词，直接问。**
- **让 Claude 采访你**（官方 Tip）："For larger features, have Claude interview you first." 用一个极简提示词开场，让 Claude 用 `AskUserQuestion` 工具采访你，问到技术实现、UI/UX、边界情况与权衡，然后写一份 spec 到 SPEC.md；**spec 写好后开一个全新会话去执行它**（干净的上下文专注实现）。
- **尽早、频繁纠偏**（官方 Tip）："Correct Claude as soon as you notice it going off track." `Esc` 中断、`Esc+Esc` 或 `/rewind` 回滚、"Undo that" 让 Claude 回退。官方给了一条硬经验：

> "If you've corrected Claude more than twice on the same issue in one session, the context is cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt that incorporates what you learned."
> （同一问题在同一会话里纠正了两次以上，说明上下文已被失败尝试污染。`/clear` 后用吸收了教训、更具体的提示词重来。）

- **主动管理上下文**（官方 Tip）："Run `/clear` between unrelated tasks to reset context." 接近上限时用 `/compact <指令>`（如 `/compact Focus on the API changes`）控制压缩方向；快速小问题用 `/btw`（答案进浮层，**永不进对话历史**）。

### 5. 官方点名的常见失败模式

| 失败模式 | 官方定义 | 官方对策 |
|---|---|---|
| **厨房水槽会话** | 一个任务做到一半插入无关问题，上下文满是无关信息 | `/clear` between unrelated tasks |
| **反复纠正** | 错了纠正、再错再纠正，上下文被失败尝试污染 | 两次纠正无效就 `/clear`，把教训写进更好的初始提示词 |
| **过度膨胀的 CLAUDE.md** | 太长导致 Claude 忽略一半，重要规则淹没在噪音里 | "Ruthlessly prune." 能删就删，或转成 hook 强制 |
| **信任而未见证** | Claude 产出看起来合理但不处理边界情况的实现 | "Always provide verification... If you can't verify it, don't ship it." |
| **无限探索** | 让 Claude "调研一下"却不限定范围，读了上百个文件 | 收敛调研范围，或用 subagent 免得污染主上下文 |

---

## 三、CLAUDE.md：把"你反复重讲的话"固化下来

### 1. 它是什么：两种跨会话记忆之一

官方文档明确：每个会话都以**全新的上下文窗口**开始，靠两种机制跨会话携带知识——

> "**CLAUDE.md files**: instructions you write to give Claude persistent context. **Auto memory**: notes Claude writes itself based on your corrections and preferences."
> （**CLAUDE.md 文件**：你写下的指令，给 Claude 持久的上下文；**自动记忆**：Claude 根据你的纠正与偏好自己记的笔记。）

| | **CLAUDE.md** | **自动记忆（auto memory）** |
|---|---|---|
| 谁写的 | 你 | Claude 自己 |
| 装什么 | 指令与规则 | 经验与模式 |
| 范围 | 项目 / 用户 / 组织 | 按仓库（worktree 间共享） |
| 加载 | 每次会话 | 每次会话（`MEMORY.md` 前 200 行或 25KB） |
| 用途 | 编码规范、工作流、项目架构 | 构建命令、调试洞见、发现的偏好 |

> "The more specific and concise your instructions, the more consistently Claude follows them."
> （指令越具体、越简洁，Claude 就越一致地遵循它们。）

### 2. 何时往 CLAUDE.md 里加内容（官方判据）

官方文档："Treat CLAUDE.md as the place you write down what you'd otherwise re-explain."（把 CLAUDE.md 当作你写下"本来要再解释一遍"的地方。）四个触发点：

- Claude 第二次犯同一个错误；
- 代码评审抓到 Claude 本应知道的代码库常识；
- 你又输入了一遍上一会话就输入过的纠正；
- 新同事上手也需要同样的背景。

### 3. 怎么写才有效：小、结构化、具体、不矛盾

官方给出的四条写作准则（Write effective instructions）：

- **大小（Size）**："target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."（**每个文件 200 行以内**；再长就吃上下文、降遵从度。）
- **结构（Structure）**：用 Markdown 标题与列表分组，像读者扫结构一样组织。
- **具体（Specificity）**：指令要具体到可验证——"Use 2-space indentation" 好于 "Format code properly"；"Run `npm test` before committing" 好于 "Test your changes"。
- **一致（Consistency）**："if two rules contradict each other, Claude may pick one arbitrarily."（两条规则互相矛盾时，Claude 可能任选一条。）定期清理冲突规则。

官方的精简判据（金句）：

> "Keep it concise. For each line, ask: *'Would removing this cause Claude to make mistakes?'* If not, cut it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"
> （保持精简。对每一行问一句"删掉它，Claude 会犯错吗？"不会就删。**臃肿的 CLAUDE.md 会让 Claude 无视你真正的指令！**）

> "You can tune instructions by adding emphasis (e.g., 'IMPORTANT' or 'YOU MUST') to improve adherence."
> （可以用强调措辞如 "IMPORTANT" / "YOU MUST" 提高遵从度。）

官方给出的**放入 / 排除**清单：

| ✅ 放入 | ❌ 排除 |
|---|---|
| Claude 猜不出的 Bash 命令 | Claude 读代码就能推断出的东西 |
| 与默认不同的代码风格规则 | Claude 本就知道的标准语言约定 |
| 测试说明与偏好的测试运行器 | 详细 API 文档（给链接即可） |
| 仓库规范（分支命名、PR 约定） | 频繁变化的信息 |
| 本项目特定的架构决策 | 长篇讲解或教程 |
| 环境怪癖（必需的 env 变量） | 逐文件的代码库描述 |
| 常见坑与非显然的行为 | "写干净代码"这类不言自明的话 |

### 4. 放在哪：位置决定作用域

| 作用域 | 位置 | 用途 | 是否共享 |
|---|---|---|---|
| 组织策略 | macOS `/Library/.../ClaudeCode/CLAUDE.md`；Linux/WSL `/etc/claude-code/CLAUDE.md`；Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | 公司级规范、安全策略 | 全组织（IT 部署） |
| 个人（所有项目） | `~/.claude/CLAUDE.md` | 个人偏好、工具习惯 | 仅自己 |
| 项目（团队共享） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目架构、编码规范、常见工作流 | 团队（入库） |
| 个人（当前项目） | `./CLAUDE.local.md` | 个人项目偏好；**记得加 `.gitignore`** | 仅自己 |

**加载机制**：Claude 从当前工作目录**向上**逐级找 `CLAUDE.md` / `CLAUDE.local.md` 并全部拼接加载（根目录在上、工作目录在下；同目录里 `CLAUDE.local.md` 排在 `CLAUDE.md` 之后）；**子目录**里的 CLAUDE.md 不在启动时加载，而是等 Claude 读到该目录文件时才按需加载。这就是 monorepo 里"分层 CLAUDE.md"的原理。

**导入语法**：`@path/to/import` 可把其他文件展开进上下文，如 `See @README for project overview and @package.json for available npm commands.`；相对路径相对**所在文件**解析；支持递归导入，**最深四层**；不想被导入时用反引号包起来（`` `@README` ``）。

**AGENTS.md**：官方明确 "Claude Code reads `CLAUDE.md`, not `AGENTS.md`." 若仓库已有 AGENTS.md，写一个 `CLAUDE.md` 导入它即可共用同一份指令：

```markdown
@AGENTS.md

## Claude Code
Use plan mode for changes under `src/billing/`.
```

### 5. 别写进 CLAUDE.md 的东西：先分清四个机制

官方 `Extend Claude Code` 文档专门给了取舍表（核心判据是"是否每次会话都要、是否要保底执行"）：

| 机制 | 加载时机 | 适用 | 例子 |
|---|---|---|---|
| **CLAUDE.md** | 每次会话自动加载 | "always do X" 规则、常驻事实 | "Use pnpm, not npm. Run tests before committing." |
| **Skill** | 按需（被调用或 Claude 判断相关时） | 参考材料、可触发的工作流 | `/deploy` 部署清单、API 风格指南 |
| **`.claude/rules/`** | 每次会话，或匹配到文件路径时 | 可按文件类型/目录定向的规则 | `src/api/**/*.ts` 的 API 开发规则 |
| **Hook** | 生命周期事件触发（脚本） | 必须每次发生、零例外 | 每次编辑后跑 eslint、禁止改 `.env` |

两条官方原话最值得记住：

> "Put it in CLAUDE.md if Claude should always know it: coding conventions, build commands, project structure, 'never do X' rules. Put it in a skill if it's reference material Claude needs sometimes..."
> （Claude 应该**始终**知道的内容放 CLAUDE.md；只是**偶尔**需要的参考材料放 skill。）

> "If a rule must hold every time, make it a hook rather than a prompt instruction."
> （如果一条规则**必须每次都生效**，写成 hook 而不是提示词指令——CLAUDE.md 是"请求"，hook 才是"强制"。）

官方还给出了"随着使用逐步加扩展"的触发表（Build your setup over time），CLAUDE.md 是第一站：

> "Claude gets a convention or command wrong twice → Add it to CLAUDE.md"
> （Claude 把某个约定/命令弄错两次 → 加进 CLAUDE.md。）

其余触发：同一提示词反复手敲 → 存成 skill；同一段多步流程第三遍粘贴 → 存成 skill；希望某件事每次必发生 → 写 hook。

### 6. 常见坑与排查

- **Claude 不遵守 CLAUDE.md**：官方直言 "there's no guarantee of strict compliance, especially for vague or conflicting instructions"（对模糊或冲突的指令尤其没有保证）。排查顺序：跑 `/context` 确认文件真的加载了 → 检查位置是否在加载范围内 → 把指令写得更具体 → 检查有没有互相矛盾的规则。**必须保底的规则写成 hook。**
- **文件太臃肿**：超 200 行就降遵从度。用 `.claude/rules/` 的 `paths` 定向加载，或把参考内容挪进 skill。`/doctor` 能给出删减建议（会砍掉 Claude 能从代码里推导的内容，保留坑、理由与偏离默认的约定）。
- **`/compact` 后指令"消失"**：项目根 CLAUDE.md **压缩后会从磁盘重新注入**；子目录里的嵌套 CLAUDE.md **不会**自动重注，要等再读到该目录文件时才会重新加载。

---

## 四、速查清单

**提示词**
- [ ] 给 Claude 一个能运行的检查（测试 / 构建 / 截图对比），并让它展示证据而非口头声称
- [ ] 复杂任务先探索、再计划、后编码；能一句话说清的 diff 就跳过计划
- [ ] 目标具体：划范围、指信息源、参考现有模式、描述症状
- [ ] 用 `@` 引用文件、贴图、给 URL、管道传入数据
- [ ] 同问题纠正两次就 `/clear`，用更好的提示词重来；无关任务之间 `/clear`

**CLAUDE.md**
- [ ] 用 `/init` 生成初版，再持续打磨；入库让团队共同维护
- [ ] 触发判据：Claude 第二次犯同一个错 / 评审抓到本应知道的常识 / 你又输入了上会话的纠正
- [ ] 每行问一句"删掉它，Claude 会犯错吗？"；**控制在 200 行以内**
- [ ] 具体到可验证："2 空格缩进"好于"格式规范点"
- [ ] 常驻规则放 CLAUDE.md，偶尔要用的放 skill，必须生效的写 hook
- [ ] 用 `/context` 确认加载、`/memory` 浏览编辑

---

## 参考来源

本文内容综合以下 Anthropic 官方文档整理（均于 2026-08-04 通过 web_fetch 从 `code.claude.com/docs` 获取，原文可直接打开核对）：

- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （agentic 定义、上下文窗口总纲、验证机制、探索→计划→编码、具体上下文、沟通与迭代、常见失败模式、CLAUDE.md 写作要点与放入/排除清单）
- **Memory: How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （CLAUDE.md 与自动记忆对比、何时添加、写作四准则、存放位置与加载机制、`@path` 导入、AGENTS.md、`.claude/rules/`、常见问题排查）
- **Extend Claude Code** — https://code.claude.com/docs/en/features-overview
  （CLAUDE.md / Skills / subagents / hooks / MCP 的取舍表、随使用逐步扩展的触发表、各特性的上下文成本）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （启动时加载内容、压缩后各机制的去留："Project-root CLAUDE.md and unscoped rules — Re-injected from disk；Nested CLAUDE.md in subdirectories — Lost until a file in that subdirectory is read again"）

> 相关仓库文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（同主题的完整扩展版，含 Skills 与失败模式详述）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（大仓库语境下的分层 CLAUDE.md 与 LSP）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
