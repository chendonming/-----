# Claude Code 权限模式（permission modes）与 auto mode：官方文档解读

> **一句话总结**：Claude Code 的权限体系是"**模式定基线 + 规则做微调**"的两层结构。官方定义了六种权限模式：`default`（Manual，默认逐项确认）、`acceptEdits`（自动接受文件编辑）、`plan`（只读探索、先出计划再动手）、`auto`（一切交给一个独立的**分类器模型**做后台安全检查）、`dontAsk`（只跑预批准的工具，面向 CI）、`bypassPermissions`（跳过全部检查，只限隔离环境）。其中 **auto mode 是官方着墨最多的模式**：它不是"无脑放行"，而是由分类器逐动作审查，默认阻止生产部署、外发敏感数据、破坏性 git 操作等；连续被拦 3 次会自动回退成提示模式。官方反复强调 auto 模式"**减少权限提示，但不保证安全**"，且 `auto` 只能设在**用户级**设置（`~/.claude/settings.json`）里做默认——仓库无法给自己授予 auto 模式。
>
> 本文基于 Claude Code 官方文档 `Choose a permission mode`、`Configure permissions`、`Configure auto mode` 三页整理，文末附参考来源。

Claude Code 里有一个高频出现的交互：Claude 想动文件、跑命令、发请求时，会停下来弹一个权限确认框。用久了你会觉得它打扰，但完全不问又怕出事。权限模式（permission modes）就是官方给"**打断频率**"做的刻度盘。

---

## 一、为什么要"权限模式"？——一个关于打断频率的开关

官方文档开头一句话就点明了它要解决的问题：

> "When Claude wants to edit a file, run a shell command, or make a network request, it pauses and asks you to approve the action. Permission modes control how often that pause happens."
> （当 Claude 想编辑文件、运行 shell 命令或发起网络请求时，它会暂停并请你批准这个动作。权限模式控制这种暂停发生的**频率**。）

接着官方给出了这套体系的整体定位——**模式是基线，规则是叠加**：

> "Modes set the baseline. Layer permission rules on top to pre-approve or block specific tools."
> （模式设定**基线**；在其上叠加权限规则，来预批准或阻止特定工具。）

这句话是理解整个权限体系的关键：你在"模式"这个维度选择"宏观上放多松"，再用 allow / ask / deny 规则在"微观上"逐个工具收紧或放行。两层可以叠加，且在 `bypassPermissions` 之外的所有模式里规则都生效。

## 二、六种模式一览

官方把六种模式整理成一张"每种模式在什么情况下**无需询问**直接运行"的对照表：

| 模式 | 无需询问就能运行什么 | 适合场景 |
|---|---|---|
| `default` | 只读 | 刚开始用、敏感工作 |
| `acceptEdits` | 读、文件编辑、常用文件系统命令（`mkdir`、`touch`、`mv`、`cp` 等） | 边写边审、你会回头看 diff |
| `plan` | 读，加上 auto mode 可用时**分类器批准的**命令 | 改动前先探索代码库 |
| `auto` | 一切，但有后台安全检查 | 长任务、减少提示疲劳 |
| `dontAsk` | 只有预先批准过的工具 | 锁死的 CI 和脚本 |
| `bypassPermissions` | 一切 | **仅限**隔离的容器和 VM |

两个容易混淆的细节：

- 每步都问的模式在 CLI 里显示为 **Manual**，但它的配置值叫 `default`——官方原文："The mode that reviews every action is named **Manual** in the CLI…Its config value is `default`."（CLI 里这个逐项审查的模式叫 Manual，它的配置值是 `default`。）
- 模式是会话级的状态，可以随时切：CLI 里按 `Shift+Tab` 循环切换，启动时用 `claude --permission-mode plan`，或者用 `permissions.defaultMode` 在设置文件里设默认值。

## 三、三种常用模式：default / acceptEdits / plan

### default（Manual）——默认逐项确认

只读操作免打扰，其余一律问。适合你还不信任当前任务方向的场景。

### acceptEdits——自动接受文件编辑

> "`acceptEdits` mode lets Claude create and edit files in your working directory without prompting. …In addition to file edits, `acceptEdits` mode auto-approves common filesystem Bash commands: `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, and `sed`."
> （`acceptEdits` 模式让 Claude 无需提示即可在**你的工作目录**里新建和编辑文件；此外还自动批准一组常用文件系统命令：`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp` 和 `sed`。）

注意两个边界：自动批准**只适用于工作目录或 `additionalDirectories` 之内**的路径；范围之外、受保护路径、以及其余所有 Bash 命令仍会弹窗。官方推荐的用法是：

> "Use `acceptEdits` when you want to review changes in your editor or via `git diff` after the fact rather than approving each edit inline."
> （当你想事后在编辑器或通过 `git diff` 审查改动、而不是逐条就地批准时，用 `acceptEdits`。）

一句话：**"先放开手，改完一起看"**。

### plan——只探索、不动手

> "Plan mode tells Claude to research and propose changes without making them. Claude reads files, runs shell commands to explore, and writes a plan, but does not edit your source."
> （Plan 模式让 Claude 做调研并**提出**改动方案，而不是真的去改。Claude 读文件、跑 shell 命令探索、写出一份计划，但**不编辑你的源码**。）

计划成型后，官方给出的批准选项也值得留意——批准计划的同时可以顺带切换模式："**Yes, and use auto mode**"（批准并直接进入 auto 模式）、"Yes, manually approve edits"（批准但逐条审查）、"No, keep planning"（继续规划）。

## 四、auto mode：官方最看重的"带安全网的长跑模式"

这是整个权限体系里官方篇幅最大的部分。它的核心机制**不是**"关闭确认"，而是把确认动作**换成一个分类器模型来执行**：

> "Auto mode lets Claude execute without routine permission prompts. A separate classifier model reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read."
> （Auto 模式让 Claude 无需常规权限提示即可执行。一个**独立的分类器模型**在动作运行前审查它们，阻止任何超出你请求范围、指向未识别基础设施、或像是被 Claude 读到的恶意内容所驱动的东西。）

### 可用性要求

auto 模式不是默认全量开放的。官方列了四类要求（任一不满足就会提示"auto mode 不可用"，且这**不是**临时故障）：

| 维度 | 要求 |
|---|---|
| 计划 | 所有套餐（All plans） |
| 组织 | Team / Enterprise 默认可用，管理员可在 managed settings 里设 `disableAutoMode` 关闭 |
| 模型 | Anthropic API 上需 Claude Opus 4.6+、Sonnet 4.6+ 或 Fable 5（部分云厂商另有限制，如仅 Sonnet 5 / Opus 4.7+） |
| 提供商 | Anthropic API、Bedrock、Vertex、Foundry 等默认可用 |

还有一个关键的**防自我授权设计**——`auto` 作为默认值只能在用户级设置里生效：

> "Claude Code v2.1.142 and later ignore `auto` from those files so a repository cannot grant itself auto mode. Move it to `~/.claude/settings.json`."
> （Claude Code v2.1.142 起忽略项目级设置文件里的 `auto`，这样**仓库无法给自己授予 auto 模式**。把它移到 `~/.claude/settings.json`。）

### 默认"阻止"与默认"放行"

分类器不是白名单式的，而是一套行为清单。官方给出了两长串默认规则，摘其要点：

**默认阻止（Blocked by default）**：

- 下载并执行代码，如 `curl | bash`
- 向外部端点发送敏感数据
- 生产环境部署与迁移（production deploys and migrations）
- 云存储上的批量删除
- 授予 IAM 或仓库权限
- 修改共享基础设施
- 破坏性 git 操作：`git reset --hard`、`git checkout -- .`、`git clean -fd`、force push、以及 `git commit --amend`（当 HEAD 提交不是本会话创建的）
- `terraform destroy` / `pulumi destroy` / `cdk destroy` 等会销毁资源的操作
- 运行带"解除安全防护"标志的命令，如 `--insecure`

**默认放行（Allowed by default）**：

- 工作目录内的本地文件操作
- 安装 lock 文件 / manifest 里声明的依赖
- 读取 `.env` 并把凭据发给对应的 API
- 只读 HTTP 请求
- 推送到**你正在工作的这个仓库**的任何分支（含默认分支）

注意这套默认规则是可查的：`claude auto-mode defaults` 会把完整的规则清单以 JSON 打印出来。

### 会话中的口头边界也算数

分类器会把**你在对话里说的话**当作边界信号：

> "The classifier treats boundaries you state in the conversation as a block signal. If you tell Claude 'don't push' or 'wait until I review before deploying', the classifier blocks matching actions even when the default rules would allow them."
> （分类器把你在对话中陈述的边界当作**阻止信号**。如果你说"别推"或"等我审完再部署"，分类器就会阻止匹配的动作，即使默认规则本来允许。）

但它也提示了这种方式的局限：口头边界不存成规则，一旦上下文压缩（context compaction）把那条消息删掉，边界就丢了。想要"硬保证"就写 deny 规则。

### 兜底机制：连续被拦就"降级回提示"

auto 模式给自己留了退路：

> "If the classifier blocks an action 3 times in a row or 20 times total, auto mode pauses and Claude Code resumes prompting. Approving the prompted action resumes auto mode."
> （如果分类器**连续 3 次**或**累计 20 次**阻止动作，auto 模式会暂停，Claude Code 恢复弹窗提示。批准该动作后 auto 模式恢复。这两个阈值不可配置。）

也就是说，如果分类器频繁误伤，说明它缺的是对你基础设施的上下文——这时该做的不是硬扛，而是去 `autoMode.environment` 里把可信的仓库/桶/域名告诉它。

### 一个容易误读的安全设计

很多人的第一反应是"分类器也是模型，会不会被文件里的恶意内容骗？"官方给出了设计答案：

> "The classifier sees user messages, tool calls, and your CLAUDE.md content. Tool results are stripped, so hostile content in a file or web page cannot manipulate it directly."
> （分类器看到的是用户消息、工具调用和你的 CLAUDE.md 内容；**工具结果被剥离**，所以文件或网页里的恶意内容无法直接操纵它。）

### 官方对 auto mode 的警告

> "Auto mode reduces permission prompts but does not guarantee safety. Use it for tasks where you trust the general direction, not as a replacement for review on sensitive operations."
> （Auto 模式减少权限提示，但**不保证安全**。把它用在你能信任大致方向的任务上，而不是敏感操作的审查替代品。）

这是官方对 auto mode 最诚实的一句定位：它解决的是"提示疲劳"，不解决"该不该做"。

## 五、两个"极端"模式：dontAsk 与 bypassPermissions

### dontAsk——面向 CI 的"只跑预批准工具"

> "If you set `dontAsk` mode, Claude Code auto-denies every tool call that would otherwise prompt you. Claude runs only actions matching your `permissions.allow` rules, read-only Bash commands, and calls approved by a PreToolUse hook. Use this mode for CI pipelines or restricted environments where you pre-define exactly what Claude may do; the session never waits for input."
> （`dontAsk` 模式下，Claude Code 自动**拒绝**所有本会弹窗的工具调用。Claude 只运行匹配你 `permissions.allow` 规则的动作、只读 Bash 命令、以及被 PreToolUse hook 批准调用。用在 CI 流水线或受限环境——那里你预先定义好 Claude 能做什么，会话**永不等待输入**。）

注意它的措辞：不是"不问"，而是"**默认拒绝**"——允许名单之外的一律拒。这与 auto 模式（默认放行 + 分类器把关）是相反的哲学。

### bypassPermissions——跳过一切，仅限隔离环境

> "`bypassPermissions` mode disables permission prompts and safety checks so tool calls execute immediately, including writes to protected paths."
> （`bypassPermissions` 模式禁用权限提示与安全检查，工具调用立即执行，包括对受保护路径的写入。）

官方对它的约束极为严格：

> "Only use this mode in isolated environments like containers, VMs, or dev containers without internet access, where Claude Code cannot damage your host system."
> （**只在**隔离环境里使用该模式：容器、VM 或无网络的 dev container——那里 Claude Code 无法损害你的宿主系统。）

并直接建议"想少弹窗又有安全网"就用 auto mode 替代它：

> "`bypassPermissions` offers no protection against prompt injection or unintended actions. For background safety checks with far fewer permission prompts, use auto mode instead."
> （`bypassPermissions` 对提示注入或非预期动作**毫无防护**。想要更少的权限提示、又有后台安全检查，请改用 auto mode。）

另外它有几个硬性护栏：`rm -rf /`、`rm -rf ~` 这类针对根目录/家目录的删除**仍然弹窗**（防模型错误的"断路器"）；Linux/macOS 上以 root/sudo 运行时直接拒绝启动；首次启用会弹一个"你需为无检查操作负责"的确认对话框。

## 六、权限规则（allow / ask / deny）：在模式之上做微调

光有模式不够，官方还提供了工具级的规则。三类规则：

> "**Allow** rules let Claude Code use the specified tool without manual approval. **Ask** rules prompt for confirmation whenever Claude Code tries to use the specified tool. **Deny** rules prevent Claude Code from using the specified tool."
> （Allow 规则：无需人工批准即可使用指定工具；Ask 规则：每次尝试使用指定工具都弹确认；Deny 规则：禁止使用指定工具。）

求值顺序是固定的，与规则写的具体程度无关：

> "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order."
> （规则按**先 deny、再 ask、后 allow** 的顺序求值，第一个命中的决定结果；规则的具体程度不改变顺序。）

这带来一个反直觉的结论：一条宽泛的 `Bash(aws *)` deny 规则能拦下所有匹配调用，即使同时存在更精确的 `Bash(aws s3 ls)` allow 规则——**deny 优先，allow 无法给 deny 开例外**。

规则语法是 `Tool` 或 `Tool(specifier)`，例如：

| 规则 | 效果 |
|---|---|
| `Bash(npm run build)` | 精确匹配该命令 |
| `Bash(git push *)` | 匹配以 `git push ` 开头的命令 |
| `Read(./.env)` | 匹配读取当前目录的 `.env` |
| `WebFetch(domain:example.com)` | 匹配抓取 example.com |
| `Agent(Explore)` | 匹配某个 subagent |

最值得记住的一句官方说明是——**规则是硬约束，不由模型决定**：

> "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."
> （权限规则由 **Claude Code 强制执行，不是由模型执行**。你提示词或 CLAUDE.md 里的指示塑造 Claude *尝试*做什么，但不改变 Claude Code 允许什么。）

设置文件还有一个优先级顺序：**managed（托管，不可覆盖）> 命令行参数 > 本地项目设置 > 共享项目设置 > 用户设置**。只要任一层 deny 了，其他层都不能放行。

## 七、auto mode 的配置：把"你的基础设施"讲给分类器听

`auto-mode-config` 文档专讲 auto mode 的调优。核心逻辑是：

> "By default, the classifier trusts only the working directory and the current repo's configured remotes. Actions like pushing to your company's source-control org or writing to a team cloud bucket are blocked until you add them to `autoMode.environment`."
> （默认情况下，分类器只信任工作目录和当前仓库已配置的 remote。推到公司代码托管组织、写团队云桶这类动作会被拦下，直到你把它们加进 `autoMode.environment`。）

于是最常用的配置是 `autoMode.environment`——用**自然语言**告诉分类器哪些是可信基础设施。官方强调条目是 prose 不是正则："Entries are prose, not regex or tool patterns. The classifier reads them as natural-language rules."（条目是自然语言描述，不是正则或工具模式；分类器把它们当作自然语言规则来读。）加 `"$defaults"` 字面量可以保留内置默认条目、在其基础上追加。

想在 auto 模式里保留"人工检查点"（比如 push / 开 PR 前必须确认），官方给的方案是内容范围的 `ask` 规则：

```json
{
  "permissions": {
    "ask": [
      "Bash(git push *)",
      "Bash(gh pr create *)"
    ]
  }
}
```

> "Content-scoped ask rules like the ones below are evaluated before the classifier and always force a permission prompt, even in auto mode, because an explicit ask rule is your stated intent to be prompted for that action."
> （内容范围的 ask 规则在分类器**之前**求值，即使在 auto 模式也**总是**强制弹权限提示——因为显式 ask 规则就是你"这个动作要问我"的明确意图。）

需要更细的自定义时，`autoMode` 还提供三个覆盖内置规则的数组，优先级从高到低：

| 配置 | 语义 | 谁能覆盖 |
|---|---|---|
| `hard_deny` | 无条件安全边界 | 用户意图和 allow 例外都无效 |
| `soft_deny` | 破坏性动作 | 用户意图、allow 例外可覆盖 |
| `allow` | 对 soft block 规则的例外 | —— |

另有两个实用能力：`autoMode.classifyAllShell: true` 让分类器审查**每一条** shell 命令（连窄 allow 规则放行的也不放过，用延迟换覆盖）；`claude auto-mode` 子命令族（`defaults` / `config` / `critique` / `reset`）可查看内置规则、核对生效配置、让 AI 评审你的自定义规则。

## 八、选型建议：按场景挑模式

把官方表述翻译成可执行建议：

| 场景 | 推荐模式 | 理由（官方口径） |
|---|---|---|
| 刚上手 / 敏感操作 / 不放心方向 | `default`（Manual） | 逐项确认，最稳 |
| 日常写码，愿意事后看 diff | `acceptEdits` | 自动接受工作目录内文件编辑，范围外仍弹窗 |
| 改动前先摸清代码库 | `plan` | 只读探索、先出计划，批准后才动手 |
| 长任务、想少被打断 | `auto` | 分类器把关 + 默认阻止危险动作，但"不保证安全" |
| CI / 受限脚本，行为完全预设 | `dontAsk` | 白名单之外一律拒绝，永不等待输入 |
| 隔离容器 / VM（无网） | `bypassPermissions` | 跳过一切检查；**勿**在能损害宿主的地方用 |

两个官方反复强调、容易踩的坑，单独拎出来：

- **`auto` ≠ 关闭安全**。它把"人工确认"换成了"分类器审查"，减少的是**频率**而非**风险**；敏感操作官方明说"不要用 auto 替代人工审查"。
- **`defaultMode: "auto"` 放错文件会静默失效**。项目级设置（`.claude/settings.json`）里的 `auto` 会被忽略——这是官方的防自我授权设计，请放到 `~/.claude/settings.json`。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 通过 web_fetch 获取）：

- **Choose a permission mode** — https://code.claude.com/docs/en/permission-modes
  （六种模式定义与对照表、切换方式、acceptEdits / plan / auto / dontAsk / bypassPermissions 各模式详解、受保护路径、auto 模式分类器机制与默认阻止/放行清单、兜底阈值）
- **Configure permissions** — https://code.claude.com/docs/en/permissions
  （分层权限体系、allow/ask/deny 规则与求值顺序、规则语法、设置优先级、managed settings、工作目录与 sandbox 的关系）
- **Configure auto mode** — https://code.claude.com/docs/en/auto-mode-config
  （autoMode.environment 可信基础设施配置、permissions.ask 人工检查点、hard_deny/soft_deny/allow 覆盖规则、classifyAllShell、`claude auto-mode` 子命令）

> 相关文档：`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（官方对大代码库的推荐姿势）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（CLAUDE.md 与上下文管理）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
