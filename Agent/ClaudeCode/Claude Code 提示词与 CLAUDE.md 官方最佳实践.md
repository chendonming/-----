# Claude Code 提示词与 CLAUDE.md 官方最佳实践

> **一句话总结**：官方对"怎么写提示词"和"怎么写 CLAUDE.md"给的建议都不是玄学，而是**一系列可以逐条对号入座、可验证的具体判据**——提示词的核心是**给 Claude 一个能运行通过的检查、先探索后计划再编码、把上下文给具体**；CLAUDE.md 的核心是**只写 Claude 从代码里推断不出的东西，每行都要问"删掉它 Claude 会犯错吗"，目标每个文件 200 行以内**。
>
> 本文以 Anthropic 官方文档 `Best practices for Claude Code`、`Memory: How Claude remembers your project`、`Prompt library` 三份资料为准整理，所有引文均为官方原文，文末附参考来源。

Claude Code 是 agentic（自主）编程环境：不是"你问一句、它答一句"的聊天机器人，而是"你描述想要什么，它探索、规划、实现"。官方对怎么用好它，给了两份核心材料——`best-practices` 讲怎么和 Claude 沟通，`memory` 讲怎么用 CLAUDE.md 把项目知识固化下来。这两件事，恰好就是大部分人上手 Claude Code 最容易困惑的两个点。

下面按官方口径逐条拆开，出处都能指到具体文档。

---

## 一、一切的起点：上下文窗口是最需要管理的资源

官方把几乎所有建议都归结到同一个约束上：

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."
> （大多数最佳实践都源于同一个约束：**Claude 的上下文窗口很快就会被占满，性能会随着它的填满而下降**。）

以及一句贯穿全文的结论：

> "The context window is the most important resource to manage."
> （**上下文窗口是最需要管理的资源**。）

上下文里装着你的整个对话：每条消息、Claude 读过的每个文件、每次命令的输出。一次调试会话就可能产生几万 token。理解了这条约束，提示词和 CLAUDE.md 的所有官方建议都变成了它的推论：给更精准的上下文、别让无意义的探索浪费空间、及时清理会话、把重复内容固化到按需加载的文件里。

---

## 二、官方对"提示词"的核心建议

以下全部来自 `Best practices for Claude Code`。官方给的建议都是"可操作的具体动作"，不是"写清楚一点"这种空话。

### 1. 给 Claude 一个能验证结果的方式

> **Tip（官方原文）**：Give Claude a check it can run: tests, a build, a screenshot to compare. It's the difference between a session you watch and one you walk away from.
> （给 Claude 一个可以运行的检查：测试、构建、或一张可对比的截图。这是"你在旁观的会话"与"你可以放手离开的会话"之间的区别。）

官方解释了为什么这条如此重要：

> "Claude stops when the work looks done. Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it."
> （Claude 在"看起来做完"时就会停下。如果没有可运行的检查，"看起来做完"就是它唯一的信号，**你就成了验证循环**——每个错误都要等你发现。）

所以你的提示词里最好带上一个能返回"通过/失败"信号的检查：测试套件、构建退出码、linter、对输出做 diff 的脚本、或与设计稿对比的浏览器截图。官方给的对照示例：

| 策略 | ❌ 之前 | ✅ 之后 |
|---|---|---|
| **提供验证标准** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing" |
| **可视化验证 UI** | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| **处理根因而非症状** | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

检查的"闸门强度"可以按需调节（官方给的阶梯）：**一条提示里**让 Claude 跑检查并迭代；**跨会话**把检查设为 `/goal` 条件；**确定性闸门**用 Stop hook 写成脚本，未通过前阻止本轮结束；**第二意见**用全新上下文的 subagent 尝试反驳结果——"干活的人不该给自己打分"。

最后还有一条贯穿始终的原则：

> "Have Claude show evidence rather than asserting success: the test output, the command it ran and what it returned, or a screenshot of the result."
> （让 Claude **展示证据**而不是口头声称成功：测试输出、它运行过的命令及返回结果、或结果截图。）

### 2. 先探索、后计划、再编码

> **Tip（官方原文）**：Separate research and planning from implementation to avoid solving the wrong problem.
> （把调研与规划同实现分开，避免解决错误的问题。）

官方推荐的流程分四步：**Explore**（`Shift+Tab` 进入 plan mode，只读探索）→ **Plan**（让 Claude 产出实现计划，`Ctrl+G` 可在编辑器里直接改）→ **Implement**（批准计划或退出计划模式，按计划编码并对照验证）→ **Commit**（让 Claude 用描述性信息提交并开 PR）。

但官方同时提醒计划模式有额外开销，并给了一条很实用的判据：

> "Planning is most useful when you're uncertain about the approach, when the change modifies multiple files, or when you're unfamiliar with the code being modified. If you could describe the diff in one sentence, skip the plan."
> （计划在你**对方案不确定、改动涉及多个文件、或不熟悉要改的代码**时最有用。**如果你能用一句话描述这个 diff，就跳过计划。**）

修 typo、加一行日志、重命名变量这种范围清楚的小改动，直接让 Claude 做就好。

### 3. 提示词要具体：给上下文，不给玄学

> **Tip（官方原文）**：The more precise your instructions, the fewer corrections you'll need.
> （指令越精确，需要的纠正就越少。）

> "Claude can infer intent, but it can't read your mind. Reference specific files, mention constraints, and point to example patterns."
> （Claude 能推断意图，但它读不了心。**引用具体文件、提及约束、指向示例模式。**）

官方给的四个策略及其前后对照：

| 策略 | ❌ 之前 | ✅ 之后 |
|---|---|---|
| **划定任务范围** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **指向信息源** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **参考现有模式** | "add a calendar widget" | "look at how existing widgets are implemented on the home page to understand the patterns. HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget that lets the user select a month and paginate forwards/backwards to pick a year. build from scratch without libraries other than the ones already used in the codebase." |
| **描述症状** | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

注意官方的平衡态度：

> "Vague prompts can be useful when you're exploring and can afford to course-correct."
> （模糊的提示词在你**正在探索、能承受纠偏成本**时很有用。）

### 4. 提供富内容（rich content）

> **Tip（官方原文）**：Use `@` to reference files, paste screenshots/images, or pipe data directly.
> （用 `@` 引用文件，粘贴截图/图片，或直接把数据管道传进来。）

官方列了五种喂数据的方式：**`@` 引用文件**（Claude 会在响应前先读它，而不是你描述代码在哪）、**直接粘贴图片**、**给文档/API 参考的 URL**、**管道传入**（`cat error.log | claude`）、或者**干脆让 Claude 自己去取**（用 Bash 命令、MCP 工具、读文件）。

### 5. 让 Claude 采访你（适合较大的功能）

> **Tip（官方原文）**：For larger features, have Claude interview you first. Start with a minimal prompt and ask Claude to interview you using the `AskUserQuestion` tool.
> （对较大的功能，先让 Claude 采访你。用一个极简提示词开场，让 Claude 用 `AskUserQuestion` 工具采访你。）

官方给的开场模板（把 `[brief description]` 换成你的功能）：

> "I want to build [brief description]. Interview me in detail using the AskUserQuestion tool. Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs. Don't ask obvious questions, dig into the hard parts I might not have considered. Keep interviewing until we've covered everything, then write a complete spec to SPEC.md."

采访产出 spec 之后的官方建议很反直觉但很重要：

> "Once the spec is complete, start a fresh session to execute it. The new session has clean context focused entirely on implementation, and you have a written spec to reference."
> （spec 写好后，**另开一个全新会话去执行它**——干净的上下文专注于实现，你手里还有一份可参考的书面 spec。）

### 6. 像问资深工程师一样问问题

学习陌生代码库时，官方说直接问那些你会问同事的问题就行，**不需要特殊提示词**：日志是怎么工作的？怎么新建一个 API 端点？`foo.rs` 第 134 行的 `async move {}` 是做什么的？`CustomerOnboardingFlowImpl` 处理了哪些边界情况？

### 7. 尽早、频繁地纠偏，主动管理上下文

> **Tip（官方原文）**：Correct Claude as soon as you notice it going off track.
> （发现 Claude 偏离方向就立即纠正。）

官方给了一套快捷键：`Esc` 中断（上下文保留，可重定向）、`Esc + Esc` 或 `/rewind` 回滚、"Undo that" 让 Claude 回退改动、`/clear` 在无关任务之间重置上下文。最关键的一条纠正判据：

> "If you've corrected Claude more than twice on the same issue in one session, the context is cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt that incorporates what you learned. A clean session with a better prompt almost always outperforms a long session with accumulated corrections."
> （如果同一会话里就同一问题纠正了 Claude 两次以上，说明上下文已被失败尝试污染。**`/clear` 后带着吸收教训、更具体的提示词重来**——干净会话 + 更好的提示词，几乎总是胜过带一堆累积纠偏的长会话。）

主动管理上下文的官方手段还包括：`/clear` 频繁重置、接近上限时 Claude 自动压缩（可用 `/compact <指令>` 控制方向，并可在 CLAUDE.md 里写压缩偏好，如"压缩时始终保留修改文件清单和测试命令"）、以及用 `/btw` 问**永不进入对话历史**的快速小问题。

### 8. 官方自带提示词库：直接抄

官方有一个专门的 `Prompt library`（https://code.claude.com/docs/en/prompt-library），定位一句话：

> "Copy-paste prompts for Claude Code, tagged by task and role."
> （供 Claude Code 复制粘贴的提示词，按任务和角色打标。）

它不是按"角色扮演"分类，而是按**软件开发生命周期（SDLC）阶段**组织——`discover`（上手/理解）、`design`（设计）、`build`（构建）、`ship`（发布/交付）、`operate`（运维/运营），再细分到具体任务类别：Onboard、Understand、Plan、Prototype、Implement、Test、Debug、Review、Refactor、Steer、Git、Automate、Release、Incident、Data。每条提示词用 `{slot}` 占位符标出要你填的变量，例如：

- `give me an overview of this codebase: architecture, key directories, and how the pieces connect`（上手新代码库）
- `what would break if I deleted {target}?`（评估改动影响面）
- `I want to build {feature}. interview me about implementation, UX, edge cases, and tradeoffs until we have covered everything, then write the spec to SPEC.md`（与上文第 5 条同一个模板）
- `implement this design, then take a screenshot of the result, compare it to the original, and fix any differences`（验证 UI 改动）
- `read issue #{issue}, implement the fix, and run the tests`（修 issue）

写提示词没头绪时，先从库里挑一条改改用——它们本身就是官方推荐的"标准写法"。

### 9. 常见的失败模式

`best-practices` 最后列了五个高频错误，官方称之为"提前识别能省时间"：

| 失败模式 | 官方描述 | 官方对策 |
|---|---|---|
| **厨房水槽会话**（kitchen sink session） | 一个任务做到一半又插入无关问题，上下文全是无关信息 | 无关任务之间 `/clear` |
| **反复纠正**（correcting over and over） | 错了纠正、再错再纠正，上下文被失败尝试污染 | 两次纠正无效就 `/clear`，把教训写进更好的初始提示词 |
| **过度膨胀的 CLAUDE.md**（over-specified CLAUDE.md） | "If your CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise."（太长导致 Claude 忽略一半，重要规则淹没在噪音里） | 无情精简；删掉或转成 hook 强制 |
| **信任而未验证**（trust-then-verify gap） | Claude 产出看似合理但不处理边界情况的实现 | 始终提供验证（测试、脚本、截图）；无法验证就别发布 |
| **无限探索**（infinite exploration） | 不限定范围就让它"调研"，Claude 读了几百个文件灌满上下文 | 收敛调研范围，或用 subagent 让探索不消耗主上下文 |

---

## 三、CLAUDE.md：官方建议怎么写

以下全部来自 `Memory: How Claude remembers your project`。官方的核心框架先把两种跨会话记忆机制分清楚：

> "Each Claude Code session begins with a fresh context window. Two mechanisms carry knowledge across sessions: **CLAUDE.md files**: instructions you write to give Claude persistent context; **Auto memory**: notes Claude writes itself based on your corrections and preferences."
> （每个会话都从空的上下文窗口开始。有两种机制把知识带进新会话：**CLAUDE.md 文件**——你写的持久指令；**自动记忆（auto memory）**——Claude 根据你的纠正和偏好自己记的笔记。）

一个重要前提（在 `memory` 和 `best-practices` 里都强调）：**CLAUDE.md 是"语境"不是"强制配置"**。

> "Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead."
> （Claude 把它们当作语境，而不是强制执行的配置。想无论 Claude 怎么决定都阻止某个动作，要用 PreToolUse hook。）

### 1. 起步：先 `/init`，再持续打磨

> **Tip（官方原文）**：Run `/init` to generate a starter CLAUDE.md file based on your current project structure, then refine over time.
> （运行 `/init` 基于项目结构生成 CLAUDE.md 初版，再持续打磨。）

> "CLAUDE.md is a special file that Claude reads at the start of every conversation. Include Bash commands, code style, and workflow rules. This gives Claude persistent context it can't infer from code alone."
> （CLAUDE.md 是每个会话开始时 Claude 都会读的特殊文件。装 Bash 命令、代码风格、工作流规则——这给了 Claude 光看代码推断不出来的持久上下文。）

`memory` 文档补充了一个细节：如果 CLAUDE.md 已存在，`/init` 会**建议改进而非覆盖**。

### 2. 什么时候该往 CLAUDE.md 里加

官方判据非常具体：

> "Treat CLAUDE.md as the place you write down what you'd otherwise re-explain. Add to it when:
> - Claude makes the same mistake a second time
> - A code review catches something Claude should have known about this codebase
> - You type the same correction or clarification into chat that you typed last session
> - A new teammate would need the same context to be productive"
> （把 CLAUDE.md 当作"你本来要反复解释的东西"的存放处。出现以下情况时加进去：**Claude 第二次犯同一个错误；代码评审抓到了 Claude 本应知道的项目常识；你又输入了一遍上一会话就输入过的纠正；一个新同事上手也需要同样的背景。**）

`best-practices` 里的精简判据则是反向的"逐行审查"：

> "Keep it concise. For each line, ask: 'Would removing this cause Claude to make mistakes?' If not, cut it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"
> （保持精简。对每一行问一句："**删掉它，Claude 会犯错吗？**"不会就删。臃肿的 CLAUDE.md 会让 Claude 无视你的真正指令！）

### 3. 放在哪：作用域决定加载方式

`memory` 文档给了完整的位置表（按加载顺序从广到窄）：

| 作用域 | 位置 | 用途 | 共享给谁 |
|---|---|---|---|
| **托管策略**（managed policy） | macOS：`/Library/Application Support/ClaudeCode/CLAUDE.md`；Linux/WSL：`/etc/claude-code/CLAUDE.md`；Windows：`C:\Program Files\ClaudeCode\CLAUDE.md` | 组织级指令，IT/DevOps 统一管理 | 组织内所有用户 |
| **用户指令**（user） | `~/.claude/CLAUDE.md` | 个人在所有项目的偏好 | 仅自己（所有项目） |
| **项目指令**（project） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享的项目规范 | 团队成员（入库） |
| **本地指令**（local） | `./CLAUDE.local.md` | 个人项目内偏好；**记得加进 `.gitignore`** | 仅自己（当前项目） |

加载规则（`memory` 原文要点）：从工作目录向上逐级找 `CLAUDE.md` 和 `CLAUDE.local.md`，全部拼接进上下文而非互相覆盖；子目录里的 CLAUDE.md **不随会话启动加载**，而是 Claude 读到该目录文件时才按需加载。用 `/context` 确认文件确实加载了（在 **Memory files** 列表里查）。

### 4. 怎么写才有效：四条硬标准

`memory` 文档 "Write effective instructions" 一节给了四条可执行的标准：

| 标准 | 官方原文要点 | 示例 |
|---|---|---|
| **长度** | "target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."（每个文件目标 **200 行以内**；更长会多耗上下文、降低服从度） | — |
| **结构** | "use markdown headers and bullets to group related instructions. Claude scans structure the same way readers do."（用标题和列表分组，Claude 和读者一样扫结构） | 用 `#`、`-` 组织，别写长段落 |
| **具体性** | "write instructions that are concrete enough to verify."（写到可验证的具体程度） | "Use 2-space indentation" 好于 "Format code properly"；"Run `npm test` before committing" 好于 "Test your changes" |
| **一致性** | "if two rules contradict each other, Claude may pick one arbitrarily."（两条规则互相矛盾时，Claude 可能随意挑一条） | 定期清理冲突/过期规则 |

`best-practices` 还有一句"把 CLAUDE.md 当代码对待"的比喻：

> "Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts."
> （把 CLAUDE.md 当代码对待：出问题时审查它、定期修剪、通过观察 Claude 的行为是否真的改变来验证改动。）

官方还提到：可以用强调词（"IMPORTANT"、"YOU MUST"）提高服从度；**把 CLAUDE.md 提交进 git 让团队共建**——"The file compounds in value over time"（文件会随时间复利增值）。

### 5. 放什么、不放什么

`best-practices` 给了一张可直接照抄的对照表：

| ✅ 放入 | ❌ 排除 |
|---|---|
| Claude 猜不出的 Bash 命令 | Claude 读代码就能推断出的东西 |
| 与默认不同的代码风格规则 | Claude 本就知道的标准语言约定 |
| 测试说明与偏好的测试运行器 | 详细 API 文档（给链接即可） |
| 仓库规范（分支命名、PR 约定） | 频繁变化的信息 |
| 本项目特定的架构决策 | 长篇讲解或教程 |
| 环境怪癖（必需的 env 变量） | 逐文件的代码库描述 |
| 常见坑与非显然的行为 | "写干净代码"这类不言自明的话 |

`memory` 文档还有一个补充判据：多步骤流程、或只对代码库某一部分有用的事，**别塞 CLAUDE.md，改用 skill 或路径规则**（见下文第 8 节）。

### 6. 导入其他文件：`@path` 语法

> "CLAUDE.md files can import additional files using `@path/to/import` syntax. Imported files are expanded and loaded into context at launch alongside the CLAUDE.md that references them."
> （CLAUDE.md 可以用 `@path/to/import` 语法导入其他文件，导入内容在会话启动时被展开并加载。）

官方给的示例：

```markdown CLAUDE.md
See @README for project overview and @package.json for available npm commands for this project.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

注意细节：相对路径相对于**包含该导入的文件**解析；导入可递归、**最深四层**；想提到路径但**不导入**时用反引号包住（`` `@README` `` 是字面量）；个人且不入库的偏好用 `CLAUDE.local.md` 加进 `.gitignore`。跨 git worktree 想共享个人指令，改从主目录导入：`@~/.claude/my-project-instructions.md`。

### 7. AGENTS.md 兼容

如果仓库已用 `AGENTS.md` 服务其他编码 agent，官方说 Claude Code **只认 `CLAUDE.md`**，但可以在 CLAUDE.md 里导入 AGENTS.md，两套工具读同一份指令不重复：

> "Claude Code reads `CLAUDE.md`, not `AGENTS.md`."

```markdown CLAUDE.md
@AGENTS.md

## Claude Code
Use plan mode for changes under `src/billing/`.
```

也可以用符号链接 `ln -s AGENTS.md CLAUDE.md`（Windows 需要管理员权限或开发者模式，官方建议用 `@AGENTS.md` 导入）。

### 8. 大型项目：`.claude/rules/` 路径规则

`memory` 文档新增的章节：项目大了，可以把指令拆成多个文件放进 `.claude/rules/`，并且支持**按路径作用域加载**：

> "For larger projects, you can organize instructions into multiple files using the `.claude/rules/` directory. This keeps instructions modular and easier for teams to maintain. Rules can also be scoped to specific file paths, so they only load into context when Claude works with matching files, reducing noise and saving context space."
> （大项目可以把指令拆成多个文件放 `.claude/rules/` 目录，保持模块化、方便团队维护。规则还可以**作用域到特定文件路径**，只在 Claude 处理匹配文件时才加载，减少噪音、节省上下文。）

带 `paths` frontmatter 的规则示例（只对 `src/api/` 下的 TS 文件生效）：

```markdown .claude/rules/api-design.md
---
paths:
  - "src/api/**/*.ts"
---
- All API endpoints must include input validation
- Use the standard error response format
```

官方在 `best-practices` 里也给了"CLAUDE.md vs 其他机制"的选型原则：

> "CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand without bloating every conversation."
> （CLAUDE.md 每个会话都加载，所以只装广泛适用的东西。只在某些时候相关的领域知识或流程，改用 skill——Claude 按需加载，不撑大每个会话。）

### 9. 自动记忆（auto memory）：Claude 自己记的笔记

`memory` 文档开篇就把 CLAUDE.md 和 auto memory 的差异列成了表：

| | **CLAUDE.md 文件** | **自动记忆（auto memory）** |
|---|---|---|
| **谁写的** | 你 | Claude |
| **装什么** | 指令与规则 | 经验与模式 |
| **作用域** | 项目 / 用户 / 组织 | 按仓库（worktree 共享） |
| **何时加载** | 每次会话 | 每次会话（`MEMORY.md` 前 200 行或 25KB） |
| **用途** | 编码规范、工作流、项目架构 | 构建命令、调试洞见、发现的偏好 |

auto memory **默认开启**，存放在 `~/.claude/projects/<project>/memory/`，`MEMORY.md` 是索引。两者的使用场景官方一句话分清：

> "Use CLAUDE.md files when you want to guide Claude's behavior. Auto memory lets Claude learn from your corrections without manual effort."
> （想**主动引导** Claude 的行为就用 CLAUDE.md；想让 Claude **从你的纠正里自动学习**就用 auto memory。）

实际操作里的切换也直白：你说"always use pnpm, not npm"这类话，Claude 会存进 auto memory；想进 CLAUDE.md 就明确说"add this to CLAUDE.md"，或用 `/memory` 打开文件自己编辑。

### 10. 排查：Claude 不听话时先查加载

`memory` 文档的 troubleshooting 一节给了一条很容易被忽略的关键事实：

> "CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there's no guarantee of strict compliance, especially for vague or conflicting instructions."
> （CLAUDE.md 的内容是作为系统提示词**之后的用户消息**交付的，不是系统提示词本身。Claude 会读并尝试遵守，但**没有严格服从的保证**，尤其当指令含糊或互相冲突时。）

排查顺序（官方给的四步）：① `/context` 在 **Memory files** 列表确认文件确实加载了，文件没出现在那里就等于 Claude 看不见；② 确认 CLAUDE.md 放在会加载的位置；③ 把指令写得更具体；④ 检查是否有互相冲突的指令。如果指令必须在某个固定点执行（如每次提交前），改用 **hook**（shell 命令在固定生命周期事件触发，无论 Claude 决定如何都会执行）。CLAUDE.md 太大时，用 `/doctor` 它会提出修剪建议——剪掉 Claude 能自己从代码推导的内容，保留坑、理由和与默认不同的约定。

---

## 四、从官方口径到一张可执行的对照表

把官方两份文档的意思收拢成"什么时候用什么"：

| 场景 | 官方建议 | 出处 |
|---|---|---|
| 想让 Claude 放手做完一个任务 | 提示词里给一个能运行通过的检查（测试/构建/截图对比） | best-practices §Give Claude a way to verify its work |
| 不确定方案、改动跨多文件 | 先进 plan mode 探索→计划→编码 | best-practices §Explore first, then plan, then code |
| 能用一句话说清改动 | 跳过计划，直接做 | 同上 |
| 想让提示词更精准 | 引用具体文件、约束、示例模式；必要时让 Claude 采访你 | best-practices §Provide specific context / §Let Claude interview you |
| 同一问题纠正了两次以上 | `/clear` 后带更好的提示词重来 | best-practices §Manage your session |
| 写提示词没头绪 | 从 Prompt library 抄一条改 | prompt-library |
| 想让项目知识跨会话保留 | 用 CLAUDE.md；/init 起步，逐行精简到 200 行内 | memory §Write effective instructions |
| 只对部分文件/偶尔需要的知识 | 用 `.claude/rules/` 路径规则或 skill，别塞 CLAUDE.md | memory §Organize rules with `.claude/rules/` |
| 想让 Claude 自己积累经验 | 开 auto memory（默认开），别手动写 | memory §Auto memory |
| 指令必须无条件执行 | 用 hook，别指望 CLAUDE.md 强制 | memory §Troubleshoot / best-practices §Configure your environment |

## 速查清单

- [ ] 提示词给了"完成标准"：测试、构建、截图对比，能跑出通过/失败
- [ ] 复杂任务：先探索、后计划、再编码；一句话能说清的改动跳过计划
- [ ] 上下文具体：文件路径、报错、约束、现有模式；用 `@` 引用而非描述
- [ ] 同问题纠正两次以上就 `/clear`，把教训写进更好的初始提示词
- [ ] CLAUDE.md 用 `/init` 起步；每行问"删掉它会犯错吗"，不会就删
- [ ] CLAUDE.md 目标 200 行以内；标题+列表结构；指令可验证、不互相矛盾
- [ ] 只放 Claude 猜不出的（Bash 命令、非默认约定、坑），不放它读代码就能推断的
- [ ] 大型项目把指令拆进 `.claude/rules/`，按 `paths` 作用域加载
- [ ] 用 `/context` 验证加载、`/memory` 编辑记忆、`/doctor` 提出修剪建议
- [ ] 理解 CLAUDE.md 是"语境"不是"强制配置"，必须无条件执行的动作用 hook

---

## 参考来源

本文内容综合以下 Anthropic 官方文档整理（均于 2026-08-04 通过 web_fetch 获取）：

- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （本文第二部分主干：验证机制、探索→计划→编码、具体上下文、富内容、Claude 采访你、会话管理、失败模式；第三部分 CLAUDE.md 起步/精简/包含排除表/CLAUDE.md vs skills）
- **Memory: How Claude remembers your project** — https://code.claude.com/docs/en/memory
  （本文第三部分主干：CLAUDE.md vs auto memory、何时加、位置表、四条写作标准、`@` 导入、AGENTS.md 兼容、`.claude/rules/`、auto memory、troubleshooting）
- **Prompt library** — https://code.claude.com/docs/en/prompt-library
  （第二部分第 8 节：提示词库的定位、SDLC 阶段分类、示例提示词）

> 相关仓库文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（提示词的实用向展开，含 Skills 部分）、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 在上下文管理中的角色）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（大代码库与上下文）。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
