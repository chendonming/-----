# Claude Code 权限模式（permission modes）官方是怎么说的？

> **一句话总结**：Claude Code 的权限模式是"官方给你的一套**打扰频率开关**"——`default`（Manual）逐动作审批、`acceptEdits` 自动放行文件编辑、`plan` 先分析后动手、`auto` 靠一个**独立分类器**在后台把关并基本不问人、`dontAsk` 只放行预批准工具（适合 CI）、`bypassPermissions` 跳过一切检查（仅限隔离环境）。官方明确说 `auto` **减少提示但**"**does not guarantee safety**"，`bypassPermissions` **只应在容器/VM 等隔离环境里用**。模式是基线，可以在其上再叠 allow/ask/deny 规则；受保护的路径（`.git`、`.claude` 等）除 `bypassPermissions` 外永远不会被自动批准。
>
> 本文基于 Claude Code 官方文档 `Choose a permission mode`、`Configure auto mode`，文末附参考来源。

当 Claude 想编辑文件、跑命令、发网络请求时，它会停下来请你批准。官方对权限模式（permission modes）的定义只有一句话：

> "Control whether Claude asks before editing files or running commands. Cycle modes with Shift+Tab in the CLI or use the mode selector in VS Code, Desktop, and claude.ai."
> （控制 Claude 在编辑文件或运行命令前**是否询问**。CLI 里用 Shift+Tab 循环切换，或使用 VS Code、桌面端和 claude.ai 里的模式选择器。）

权限模式本质是一个"审查频率"旋钮，不是安全边界。官方定位如下：

> "The mode you pick shapes the flow of a session: Manual mode has you review each action as it comes, while looser modes let Claude work in longer uninterrupted stretches and report back when done."
> （你选的模式决定了会话的节奏：Manual 模式让你逐个审查动作；宽松的模式让 Claude 长时间连续工作、完成后汇报。）

---

## 一、一张表看懂六种模式

官方用一张表总结了每个模式"**不问就放行什么**"以及"**最适合什么**"：

| 模式 | 不问就执行 | 最适合 |
|---|---|---|
| `default`（界面里叫 **Manual**） | 只读 | 入门、敏感工作 |
| `acceptEdits` | 读、文件编辑、常见文件命令（`mkdir`/`touch`/`mv`/`cp` 等） | 你会在事后审查的编码迭代 |
| `plan` | 读，加上 auto mode 可用时**分类器放行的命令** | 改动前探索代码库 |
| `auto` | 一切，但带**后台安全检查** | 长任务、减少提示疲劳 |
| `dontAsk` | 只放行**预先批准**的工具 | 锁死的 CI 和脚本 |
| `bypassPermissions` | 一切 | 仅限隔离的容器/VM |

> 注意：界面里的"Manual"对应配置值 `default`（hook 和 SDK 集成用的就是 `default`）。CLI 里 `manual` 是 `default` 的别名（要求 Claude Code v2.1.200+）。

模式只是**基线**，还可以在上面叠规则：

> "Modes set the baseline. Layer permission rules on top to pre-approve or block specific tools. These controls apply in every mode, including `bypassPermissions`."
> （模式设定基线。在它之上叠权限规则，可预批准或阻止特定工具。这些控制在**包括 `bypassPermissions` 在内的每个模式**下都生效。）

## 二、三种常用模式：default / acceptEdits / plan

### 1. `default`（Manual）：逐动作审批

每件事都问，什么都不自动放行（只读除外）。适合敏感工作或刚接触时。

### 2. `acceptEdits`：自动批准文件编辑

自动放行工作目录内的文件新建/编辑，外加常见文件系统命令：

> "In addition to file edits, `acceptEdits` mode auto-approves common filesystem Bash commands: `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, and `sed`."
> （除了文件编辑，`acceptEdits` 还自动放行常见文件系统命令：`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp` 和 `sed`。）

但注意**边界**：自动放行只作用于工作目录内（或 `additionalDirectories`）；目录外、受保护路径、以及这些命令之外的其他命令仍然会询问。官方给出它的适用场景：

> "Use `acceptEdits` when you want to review changes in your editor or via `git diff` after the fact rather than approving each edit inline."
> （当你更愿意事后在编辑器里或通过 `git diff` 审查改动、而不是逐个内联批准时，用 `acceptEdits`。）

### 3. `plan`：先分析，批准后才动手

> "Plan mode tells Claude to research and propose changes without making them."
> （Plan 模式让 Claude 只做研究并提出改动方案，而**不实际执行**。）

`plan` 模式下编辑被锁定，直到你批准方案。批准时可以选：

- **Yes, and use auto mode**：批准并切到 auto mode 继续（auto 不可用则显示"auto-accept edits"）
- **Yes, manually approve edits**：批准后逐条手动批准编辑
- **No, keep planning**：留在 plan 模式继续改方案

批准后退出 plan 模式，Claude 开始动手。用 `Shift+Tab` 可随时再进/出 plan 模式。

## 三、auto mode：官方重点，用分类器消除提示

### 它是什么

> "Auto mode lets Claude execute without routine permission prompts. A separate classifier model reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read."
> （Auto mode 让 Claude 在没有常规权限提示的情况下执行。一个**独立的分类器模型**在动作执行前审查它们，阻止任何超出你请求范围、指向未识别基础设施、或看起来由 Claude 读到的恶意内容驱动的动作。）

关键点：把关的**不是 Claude 自己**，而是一个单独的分类器模型（默认跑在 Claude Sonnet 5 上，不计入你的 `/model` 选择）。分类器看到的是用户消息、工具调用和 CLAUDE.md，**工具结果被剥离**——这样文件/网页里的恶意内容没法直接操纵分类器。

### 官方警告：减少提示 ≠ 保证安全

> "Auto mode reduces permission prompts but does not guarantee safety. Use it for tasks where you trust the general direction, not as a replacement for review on sensitive operations."
> （Auto mode 减少权限提示，但**不保证安全**。把它用在你信任大方向的任务上，不要用它替代对敏感操作的审查。）

### 默认拦什么、放什么（浓缩版）

| 默认**拦截** | 默认**放行** |
|---|---|
| 下载并执行代码（如 `curl \| bash`） | 工作目录内的本地文件操作 |
| 向外部端点发送敏感数据 | 按 lockfile/manifest 装依赖 |
| 生产环境部署与迁移 | 读 `.env` 并发送凭据给对应 API |
| 云端存储的大规模删除 | 只读 HTTP 请求 |
| 授予 IAM / 仓库权限 | 往当前仓库任意分支 push、建符合请求的 PR |
| 不可逆地删除会话前就存在的文件 | |
| `git reset --hard`、`git clean -fd` 等丢弃未提交改动 | |
| `terraform destroy` / `cdk destroy` / `pulumi destroy` | |
| 强制 push、`git commit --amend`（改写非本会话创建的提交） | |

（完整规则列表可用 `claude auto-mode defaults` 以 JSON 导出。）

### 可用前提

auto mode 不是谁都能用，官方列了三条硬性要求：**计划**（所有套餐）、**模型**（Anthropic API 上需 Opus 4.6+/Sonnet 4.6+/Fable 5；Bedrock/Vertex/Foundry 上需 Sonnet 5/Opus 4.7+/Fable 5）、**Provider**（默认在 Anthropic API、Bedrock 等上可用）。还有一个容易踩的坑：

> "Claude Code v2.1.142 and later ignore `auto` from those files so a repository cannot grant itself auto mode."
> （Claude Code v2.1.142 起**忽略项目/本地 settings 里的 `defaultMode: "auto"`**，防止仓库给自己授权 auto mode。）

所以想默认 auto mode，必须写进 `~/.claude/settings.json`（用户级），而不是项目里。

### 怎么配：信任基础设施 + 规则覆盖

auto mode 默认只信任**工作目录 + 当前仓库的 remote**，公司内部 push、团队云 bucket 会被拦。官方解法是 `autoMode.environment` 用**自然语言**描述你的基础设施（组织、源码控制、受信任域、云 bucket、内部包仓库、敏感数据位置……），支持 `"$defaults"` 保留内置默认项：

```json
{
  "autoMode": {
    "environment": [
      "$defaults",
      "Source control: github.example.com/acme-corp and all repos under it",
      "Trusted cloud buckets: s3://acme-build-artifacts, gs://acme-ml-datasets",
      "Trusted internal domains: *.corp.example.com, api.internal.example.com"
    ]
  }
}
```

还有三个字段覆盖内置规则，优先级从高到低：

| 字段 | 语义 | 注意 |
|---|---|---|
| `hard_deny` | 无条件拦截，用户意图也不能解除 | 写内置安全底线 |
| `soft_deny` | 拦截破坏性动作，用户明确意图可解除 | 想收紧就用它 |
| `allow` | 对 soft block 规则的**例外** | 频繁误报时放宽 |

> ⚠️ 设置任何一项**不带** `"$defaults"` 就会**整个替换**内置默认列表（`soft_deny` 会丢掉 force push、`curl | bash`、生产部署等内置规则）。要全权接管前，先 `claude auto-mode defaults` 导出内置规则。

`autoMode.classifyAllShell: true` 可让**所有** shell 命令都过分类器（默认窄 allow 规则如 `Bash(npm test)` 会绕过分类器直接放行）。

### 被拦了怎么办 + 自动回退

- 每次拦截记录在 `/permissions` 的 **Recently denied** 标签下，按 `r` 可标记重试
- 分类器**连续拦 3 次、或累计拦 20 次**，auto mode 会**自动暂停并恢复逐条询问**（阈值不可配）
- 反复误报 → 把目标加进 `autoMode.environment`，或 `/feedback` 反馈

## 四、两个特殊模式：dontAsk / bypassPermissions

### `dontAsk`：锁死的 CI 模式

> "If you set `dontAsk` mode, Claude Code auto-denies every tool call that would otherwise prompt you. Claude runs only actions matching your `permissions.allow` rules, read-only Bash commands, and calls approved by a `PreToolUse` hook."
> （`dontAsk` 模式下 Claude Code **自动拒绝**一切本会询问你的工具调用，只运行匹配 `permissions.allow` 规则的动作、只读命令、以及 PreToolUse hook 批准的调用。）

适合 CI 流水线、受限环境——**会话从不等待输入**。它不会出现在 Shift+Tab 循环里，需用 `claude --permission-mode dontAsk` 指定。

### `bypassPermissions`：跳过一切（官方强烈限定使用场景）

> "Only use this mode in isolated environments like containers, VMs, or dev containers without internet access, where Claude Code cannot damage your host system."
> （**只**在隔离环境里用——容器、VM 或没网口的 dev container，让 Claude Code 损坏不了你的宿主机。）

连受保护路径写入都放行，但两个兜底仍在：显式 ask 规则、以及 `rm -rf /`、`rm -rf ~` 这类指向根/主目录的删除**仍会提示**（作为防模型出错的熔断）。

> "`bypassPermissions` offers no protection against prompt injection or unintended actions. For background safety checks with far fewer permission prompts, use auto mode instead."
> （`bypassPermissions` 对提示注入或意外动作**毫无防护**。想要后台安全检查又少打扰，用 auto mode 替代。）

它无法从会话中途进入，必须启动时用 `--permission-mode bypassPermissions` 或 `--dangerously-skip-permissions` 开启；Linux/macOS 下**拒绝以 root/sudo 运行**。

## 五、受保护路径：永远不被自动批准

有一小撮路径（`.git`、`.config/git`、`.vscode`、`.husky`、`.claude` 以及 `.bashrc`、`.npmrc`、`.envrc` 等配置文件）**除 `bypassPermissions` 外，任何模式都不会自动批准写入**：

> "Writes to protected paths are never auto-approved except in `bypassPermissions` mode… guarding repository state and Claude's own configuration against accidental corruption."
> （对受保护路径的写入**永远不会自动批准**，除非在 `bypassPermissions` 模式——保护仓库状态和 Claude 自身配置不被意外破坏。）

在 `auto` 模式里它们会被路由给分类器；在 `default`/`acceptEdits`/`plan` 里会弹窗询问。`permissions.allow` 规则也无法预批准这些写入。

## 六、怎么切换 / 设置

| 时机 | 做法 |
|---|---|
| 会话中 | CLI 按 `Shift+Tab` 循环 `default → acceptEdits → plan`（auto/bypassPermissions 满足条件才出现）；VS Code/桌面端/网页点模式选择器 |
| 启动时 | `claude --permission-mode plan`（`-p` 非交互运行同样支持） |
| 设为默认 | settings 里 `"permissions": { "defaultMode": "acceptEdits" }` |
| 命令 | `/permissions` 查看与改权限规则；`claude auto-mode defaults/config/critique/reset` 检视 auto 配置 |

## 七、一句话选型

| 场景 | 推荐模式 | 理由 |
|---|---|---|
| 敏感工作 / 刚上手 | `default` | 每步都审 |
| 边写边改、事后 diff 审查 | `acceptEdits` | 免去内联批准，仍保目录边界 |
| 大改动前的调研 | `plan` | 不动代码，批准后执行 |
| 长任务、信得过方向 | `auto` | 分类器后台把关，官方明说"不保证安全" |
| CI / 无人值守 | `dontAsk` | 只跑预批准的动作 |
| 隔离容器里的自主运行 | `bypassPermissions` | 唯一"全放行"，官方限定场景 |

官方给出的优先级很清晰：想要"少提示 + 后台安全检查"，用 **auto**；`bypassPermissions` 是最后手段。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 通过 web_fetch 获取，入口为官方文档索引 https://code.claude.com/docs/llms.txt）：

- **Choose a permission mode** — https://code.claude.com/docs/en/permission-modes
  （六种模式的放行表、切换方式、acceptEdits/plan/auto/dontAsk/bypassPermissions 各节细节、受保护路径清单、auto mode 可用前提与分类器默认拦截/放行清单）
- **Configure auto mode** — https://code.claude.com/docs/en/auto-mode-config
  （`autoMode.environment` 信任基础设施配置、allow/soft_deny/hard_deny 优先级与 `"$defaults"` 警告、`classifyAllShell`、`claude auto-mode` 子命令、denials 审查与回退机制）
- **Official docs index** — https://code.claude.com/docs/llms.txt
  （用于定位权限相关页面；索引中另有 `permissions`、`interactive-mode`、`sandboxing` 等相邻页面未展开）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md 管理）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（官方调研类博客范例）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
