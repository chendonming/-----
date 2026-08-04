# Claude Code 权限模式（permission modes）与 auto mode：官方文档怎么说

> **一句话总结**：Claude Code 官方把"权限控制"分成**两层**——**权限模式（permission mode）定基调**：Claude 想改文件、跑命令、发网络请求时，是停下来问你、自动放行、还是直接跳过，由当前模式决定；**权限规则（permission rules）做微调**：在任意模式下用 `allow`/`ask`/`deny` 预放行或拦截具体工具，即使 `bypassPermissions` 模式下也生效。官方共列了 **6 种模式**，其中 **auto mode** 是 2025 年后的重头戏——用**独立分类器模型**在动作执行前做安全检查，无需每个动作都弹窗。本文基于官方 `Permission modes`、`Configure auto mode`、`Configure permissions` 三篇文档整理，文末附参考来源。

很多人刚接触 Claude Code 时，把权限问题简单理解成"要不要点允许"。官方文档的框架其实更立体：一套**模式（mode）**决定"问的频率有多高"，一套**规则（rules）**决定"哪些具体动作例外"。本文把官方三篇文档的口径逐条拆开。

---

## 一、权限模式是什么：官方一句话定义

`Permission modes` 文档开头给出了最直白的定义：

> "Control whether Claude asks before editing files or running commands. Cycle modes with Shift+Tab in the CLI or use the mode selector in VS Code, Desktop, and claude.ai."
> （控制 Claude 在编辑文件或运行命令前是否询问。在 CLI 里按 `Shift+Tab` 循环切换模式，或在 VS Code、桌面端、claude.ai 里用模式选择器切换。）

官方接着解释为什么要设计这一层——它管的是"暂停的频率"：

> "Permission modes control how often that pause happens."
> （权限模式控制**那个暂停发生得多频繁**。）

核心心法在后面一句，也是全文的骨架：

> "Modes set the baseline. Layer [permission rules] on top to pre-approve or block specific tools. These controls apply in every mode, including `bypassPermissions`."
> （模式设定**基线**。在其上叠加权限规则来预放行或拦截特定工具。这些控制在**每个模式**下都生效，包括 `bypassPermissions`。）

## 二、官方 6 种模式：一张表看清"免问"范围

`Permission modes` 文档有一张核心对照表，列出各模式下 Claude "不弹窗就能做什么"：

| 模式 | 不询问就能做什么 | 官方定位（Best for） |
|---|---|---|
| `default`（CLI 里叫 **Manual**） | 只读 | Getting started, sensitive work（上手、敏感工作） |
| `acceptEdits` | 读、文件编辑、常见文件系统命令（`mkdir`、`touch`、`mv`、`cp` 等） | Iterating on code you're reviewing（边审边改） |
| `plan` | 读 + 探索性命令；auto mode 可用时由分类器放行的命令 | Exploring a codebase before changing it（改之前先探索） |
| `auto` | 一切，带后台安全检查 | Long tasks, reducing prompt fatigue（长任务、减少弹窗疲劳） |
| `dontAsk` | 只有预批准的工具 | Locked-down CI and scripts（锁死的 CI 和脚本） |
| `bypassPermissions` | 一切 | Isolated containers and VMs only（仅隔离容器/虚拟机） |

两个关键细节：

**① 默认模式叫 "Manual"，配置值叫 `default`。** 官方原文：

> "The mode that reviews every action is named **Manual** in the CLI, in `claude --help`, in the VS Code and JetBrains extensions, and in the desktop app. Its config value is `default`."

**② `bypassPermissions` 被官方圈了明确红线。** 它对文件写入、`.git`、`.claude` 等受保护路径也一律跳过检查，官方警告只允许在隔离环境用：

> "Only use this mode in isolated environments like containers, VMs, or dev containers without internet access, where Claude Code cannot damage your host system."
> （**只**在没有互联网的隔离环境——容器、VM、dev container 里用，否则 Claude Code 可能破坏你的宿主机。）

即便如此，`bypassPermissions` 下仍有两个兜底：显式 `ask` 规则、以及 `rm -rf /`、`rm -rf ~` 这类"根目录/家目录删除"作为**电路熔断**依旧弹窗。

## 三、怎么切换 & 怎么设默认

官方明确说模式是"操作层"的东西，**不是聊天里让 Claude 改的**：

> "The mode is set through these controls, not by asking Claude in chat."
> （模式通过这些控件设置，**不是在聊天里让 Claude 改的**。）

- **会话中切换（CLI）**：`Shift+Tab` 循环 `default → acceptEdits → plan`，状态栏会显示 `⏸ plan mode on`、`⏵⏵ accept edits on`、`⏵⏵ auto mode on` 等徽标。`auto`、`bypassPermissions` 满足条件后也会加入循环，`dontAsk` 永远不在循环里。
- **启动时指定**：`claude --permission-mode plan`
- **设为持久默认**：settings 文件里写 `defaultMode`：

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

> "You can switch modes mid-session, at startup, or as a persistent default."
> （你可以**会话中**、**启动时**、或作为**持久默认**来切换模式。）

## 四、auto mode：用"分类器"消灭常规弹窗

auto mode 是这几篇文档里最值得细读的部分，官方专开一页 `Configure auto mode` 讲它的配置。先看它的定义：

> "Auto mode lets Claude execute without routine permission prompts. A separate classifier model reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read."
> （auto mode 让 Claude **无需常规权限弹窗**即可执行。一个**独立的分类器模型**在动作执行前审查，拦截那些**超出你请求范围**、指向**未识别基础设施**、或似乎由 Claude 读到的**恶意内容**驱动的操作。）

关键词是"**分类器（classifier）**"——它不是不检查，而是把检查从"每次问你"换成了"一个模型在后台评估"。官方给的配套警告很清醒：

> "Auto mode reduces permission prompts but does not guarantee safety."
> （auto mode 减少权限弹窗，但**不保证安全**。）

### 4.1 使用前提

auto mode 不是所有账户都能开，官方列了三条硬性要求（Plan 全套餐可用；Team/Enterprise 管理员可关；模型需 Opus 4.6/Sonnet 4.6/Fable 5 或更新）。如果 Claude Code 报 auto mode 不可用，官方明确说"这是有前提没满足，不是临时故障"：

> "If Claude Code reports auto mode as unavailable, one of these requirements is unmet; this is not a transient outage."
> （如果 Claude Code 报告 auto mode 不可用，那是某条要求没满足，**不是临时故障**。）

还有一个防滥用细节：**仓库不能给自己开 auto mode**。`defaultMode: "auto"` 写在项目 `.claude/settings.json` 里会被忽略，必须放用户级 `~/.claude/settings.json`。

### 4.2 默认拦什么、放什么

分类器有一套内置规则。**默认拦截**的典型项（部分）：

- 下载并执行代码，如 `curl | bash`
- 向外部端点发送敏感数据
- 生产部署和迁移、云存储上的批量删除
- 授权 IAM/仓库权限、修改共享基础设施
- `git reset --hard`、`git checkout -- .`、`git clean -fd` 等会丢弃未提交改动的命令
- `terraform destroy` / `pulumi destroy` / `cdk destroy` 等销毁性操作

**默认放行**的典型项（部分）：

> "Local file operations in your working directory; Installing dependencies declared in your lock files or manifests; Reading `.env` and sending credentials to their matching API; Read-only HTTP requests."
> （工作目录内的本地文件操作；安装锁文件/清单里声明的依赖；读取 `.env` 并把凭据发给对应的 API；只读 HTTP 请求。）

放行还覆盖"推送到你正在工作的仓库的任意分支"。官方把默认清单做成了可查询命令：

> "Run `claude auto-mode defaults` to print the full rule lists as JSON."
> （运行 `claude auto-mode defaults` 把完整规则列表以 JSON 打出来。）

### 4.3 告诉分类器"什么是你信任的"：`autoMode.environment`

默认情况下分类器**只信任工作目录和当前仓库的 remote**：

> "By default, the classifier trusts only the working directory and the current repo's configured remotes."
> （默认情况下，分类器只信任工作目录和当前仓库配置的 remote。）

推送到公司 Git 组织、写团队云 bucket 这类常规操作会被拦，直到你用 `autoMode.environment` 把它加进信任清单。官方说对大多数组织这是**唯一要配的字段**：

> "For most organizations, `autoMode.environment` is the only field you need to set."
> （对大多数组织，`autoMode.environment` 是**唯一需要设置的字段**。）

示例配置（`"$defaults"` 表示保留内置默认项并插入到该位置）：

```json
{
  "autoMode": {
    "environment": [
      "$defaults",
      "Source control: github.example.com/acme-corp and all repos under it",
      "Trusted cloud buckets: s3://acme-build-artifacts, gs://acme-ml-datasets",
      "Trusted internal domains: *.corp.example.com, api.internal.example.com",
      "Key internal services: Jenkins at ci.example.com, Artifactory at artifacts.example.com"
    ]
  }
}
```

注意：条目是**自然语言描述**，不是正则或工具模式。官方建议按"给新工程师介绍基础设施"的口吻写。

### 4.4 覆盖默认规则：`hard_deny` / `soft_deny` / `allow`

分类器内部按四级优先级工作：

- `hard_deny`：无条件拦截，用户意图和 `allow` 例外都无效（安全边界）
- `soft_deny`：拦截但用户明确意图可解除
- `allow`：作为 `soft_deny` 的例外放行
- 显式用户意图：消息里直接且具体地描述了该动作时，可解除剩余 soft 拦截

> "General requests don't count as explicit intent. Asking Claude to 'clean up the repo' doesn't authorize force-pushing, but asking Claude to 'force-push this branch' does."
> （笼统的请求不算明确意图。让 Claude "清理仓库"并不等于授权 force-push；但明确说"force-push 这个分支"就算。）

配置 `autoMode` 各列表时有个**大坑**必须知道：不带 `"$defaults"` 会**整表替换**内置规则。官方 `Danger` 警告：

> "If you set an array without `"$defaults"`, you discard the built-in rules for that section: `soft_deny`: every built-in soft block rule, including force push, `curl | bash`, production deploys, and auto-mode bypass."
> （如果你设置的数组不带 `"$defaults"`，就会丢弃该节的**全部内置规则**——包括 force push、`curl | bash`、生产部署等默认拦截。）

想安全地全量接管，官方给的路径是：`claude auto-mode defaults` 导出 → 拷进自己的 settings → 逐条审一遍。

## 五、模式之上还有一层：permission rules

模式管"问的频率"，规则管"具体工具放不放行"。官方在 `Configure permissions` 里定义了三类规则：

| 规则 | 官方定义 | 效果 |
|---|---|---|
| **Allow** | "let Claude Code use the specified tool without manual approval" | 免审批使用指定工具 |
| **Ask** | "prompt for confirmation whenever Claude Code tries to use the specified tool" | 每次使用都强制确认 |
| **Deny** | "prevent Claude Code from using the specified tool" | 直接禁止 |

求值顺序是固定的，**先 deny、再 ask、最后 allow**：

> "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome."
> （规则按 **deny → ask → allow** 的顺序求值，第一个命中决定结果。）

两条容易被忽略的官方口径：

**① 规则由 Claude Code 强制，不是模型自觉。** 提示词和 `CLAUDE.md` 只能塑造 Claude"想干什么"，不能改变 Claude Code"允许什么"：

> "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."
> （权限规则由 **Claude Code 强制**，不是模型。提示词或 `CLAUDE.md` 里的指令塑造 Claude 想做什么，但**不能改变** Claude Code 允许什么。）

**② 底层是一套分层权限系统。** 读操作默认免问，Bash 命令除内置只读集外要问，文件修改要问：

> "Claude Code uses a tiered permission system to balance power and safety."
> （Claude Code 用**分层权限系统**在能力与安全之间取平衡。）

## 六、落地建议：从官方口径到"我该怎么选"

| 你的场景 | 官方建议 | 关键理由 |
|---|---|---|
| 敏感工作、刚开始用 | `default`（Manual） | 每个动作都过目 |
| 在编辑器里边审边改代码 | `acceptEdits` | 事后 `git diff` 复核，而不是逐条内联批准 |
| 改代码前先探索、只出方案 | `plan` | 读文件、跑只读命令，但不改源码，批准方案后才动手 |
| 长任务、想减少弹窗疲劳 | `auto` | 分类器后台把关，但要先确认模型/账户满足前提 |
| CI、脚本、锁死环境 | `dontAsk` | 只跑预批准工具，会话永不等待输入 |
| 隔离容器/VM 里自主运行 | `bypassPermissions` | 跳过所有检查，**仅限隔离环境**，官方明确警告 |

最后收拢两个官方反复强调的要点：

- **auto mode ≠ 无检查**，它只是把"问你"换成"分类器把关"，且明确"不保证安全"；`permissions.deny` 是比分类器更硬的一道墙（deny 先于分类器求值，且不可被覆盖）。
- **模式与规则是叠加的**：`bypassPermissions` 跳过了模式层的检查，但显式 `ask` 规则、组织对 connector 工具的 `ask` 设置、MCP 工具的 `requiresUserInteraction` 标记**依然会弹窗**——这是官方给的最后一道兜底。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 获取）：

- **Choose a permission mode** — https://code.claude.com/docs/en/permission-modes
  （本文主干：权限模式定义、6 种模式对照表、Shift+Tab 切换与 `defaultMode` 设置、acceptEdits/plan/auto/dontAsk/bypassPermissions 各模式细则、受保护路径、auto mode 使用前提与分类器默认拦截/放行清单）
- **Configure auto mode** — https://code.claude.com/docs/en/auto-mode-config
  （auto mode 配置参考：`autoMode.environment` 信任清单、`hard_deny`/`soft_deny`/`allow` 四级优先级与 `"$defaults"` 陷阱、`claude auto-mode defaults/config/reset` 子命令、`permissions.ask` 人工检查点）
- **Configure permissions** — https://code.claude.com/docs/en/permissions
  （权限规则体系：allow/ask/deny 定义与求值顺序、分层权限系统、规则由 Claude Code 强制、`permissions.defaultMode`、managed settings 与企业管控）

> 相关文档：`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent 可配置 `permissionMode` 与工具约束，权限体系的另一面）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
