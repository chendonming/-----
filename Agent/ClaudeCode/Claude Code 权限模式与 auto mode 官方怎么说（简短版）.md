# Claude Code 权限模式（permission modes）与 auto mode：官方怎么说（简短版）

> **一句话总结**：Claude Code 的权限控制是"**模式定基线 + 规则做微调**"两层结构。官方定义了六种模式——`default`（CLI 里叫 **Manual**，逐动作审批）、`acceptEdits`（自动放行工作目录内文件编辑）、`plan`（只读探索、先出方案再动手）、`auto`（一个**独立分类器模型**在后台把关，基本不弹窗）、`dontAsk`（只放行预批准工具，面向 CI）、`bypassPermissions`（跳过一切，**仅限隔离容器/VM**）。其中 auto mode 是官方着墨最多的：它**不是"无脑放行"**，而是把"问你"换成"分类器审查"，且官方明说它"**减少提示，但不保证安全**"。本文是浓缩版，基于官方 `Choose a permission mode`、`Configure auto mode`、`Configure permissions` 三篇文档，文末附来源。
>
> 知识库另有三篇同主题详细版：`Claude Code 权限模式与 auto mode 官方解读.md`、`Claude Code 权限模式与 auto mode 官方怎么说.md`、`Claude Code 权限模式（permission modes）官方说明.md`，需要展开细节可看那几篇。

---

## 一、权限模式是什么

官方定义只有一句话：

> "Control whether Claude asks before editing files or running commands."
> （控制 Claude 在编辑文件或运行命令前**是否询问**。）

它管的是"打断频率"：

> "Permission modes control how often that pause happens."
> （权限模式控制那个"停下来等你批准"的暂停**发生得多频繁**。）

权限体系的分层逻辑是全文骨架：

> "Modes set the baseline. Layer permission rules on top to pre-approve or block specific tools. These controls apply in every mode, including `bypassPermissions`."
> （模式设定**基线**；在其上叠加权限规则，来预批准或阻止特定工具。这些控制在**包括 `bypassPermissions` 在内的每个模式**下都生效。）

## 二、六种模式一张表

官方用一张表总结各模式"不弹窗就能做什么"和"适合什么"：

| 模式 | 不询问就能执行 | 官方定位（Best for） |
|---|---|---|
| `default`（CLI 里叫 **Manual**） | 只读 | 上手、敏感工作 |
| `acceptEdits` | 读、文件编辑、常用文件命令（`mkdir`/`touch`/`mv`/`cp`/`sed` 等） | 边写边审、事后看 diff 的迭代 |
| `plan` | 读，加上 auto 可用时**分类器放行**的命令 | 改动前探索代码库 |
| `auto` | 一切，带**后台安全检查** | 长任务、减少弹窗疲劳 |
| `dontAsk` | 只有**预先批准**的工具 | 锁死的 CI 和脚本 |
| `bypassPermissions` | 一切 | **仅限**隔离容器/VM |

两个容易混淆的细节：界面里的 "Manual" 对应配置值 `default`；`bypassPermissions` 对提示注入毫无防护，官方明确建议"想要少弹窗又有安全网，用 auto mode 替代它"。

## 三、auto mode：把"问你"换成"分类器审查"

官方对 auto mode 的定义，关键词是"**独立的分类器模型**"：

> "Auto mode lets Claude execute without routine permission prompts. A separate classifier model reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read."
> （Auto 模式让 Claude 无需常规权限弹窗即可执行。一个**独立的分类器模型**在动作运行前审查它们，阻止任何超出你请求范围、指向未识别基础设施、或像是被 Claude 读到的恶意内容所驱动的动作。）

它对"安全"的定位极其诚实：

> "Auto mode reduces permission prompts but does not guarantee safety. Use it for tasks where you trust the general direction, not as a replacement for review on sensitive operations."
> （Auto 模式**减少权限提示，但不保证安全**。把它用在你信任大方向的任务上，而不是敏感操作的审查替代品。）

几个关键事实：

- **默认拦截**（部分）：下载并执行代码（`curl | bash`）、向外部端点发敏感数据、生产部署/迁移、`git reset --hard`/`git clean -fd` 等丢弃未提交改动的操作、`terraform destroy`/`cdk destroy` 等销毁性操作、带 `--insecure` 之类的命令。
- **默认放行**（部分）：工作目录内的本地文件操作、按 lockfile/manifest 装依赖、读 `.env` 并发送凭据给对应 API、只读 HTTP 请求、推送到**当前工作仓库**的任意分支。
- **默认只信任**工作目录 + 当前仓库的 remote：推到公司代码托管组织、写团队云桶会被拦，直到在 `autoMode.environment` 里用**自然语言**声明它们可信（官方说对大多数组织这是唯一要配的字段）。
- **兜底**：分类器连续拦 3 次或累计 20 次，auto 模式自动暂停、恢复逐条弹窗（阈值不可配）。
- **防自我授权**：`defaultMode: "auto"` 写在项目级 `.claude/settings.json` 里会被忽略（v2.1.142+），必须放用户级 `~/.claude/settings.json`——仓库不能给自己授予 auto 模式。

## 四、规则层：allow / ask / deny

模式之上还能叠工具级规则，定义与求值顺序都很直白：

> "**Allow** rules let Claude Code use the specified tool without manual approval. **Ask** rules prompt for confirmation whenever Claude Code tries to use the specified tool. **Deny** rules prevent Claude Code from using the specified tool."
> （Allow 免审批使用指定工具；Ask 每次使用都强制确认；Deny 直接禁止。）

> "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order."
> （规则按**先 deny、再 ask、后 allow** 的顺序求值，第一个命中决定结果；规则写得具体与否不改变顺序。）

最值得记住的一句：**规则由 Claude Code 强制，不由模型决定**。

> "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."
> （权限规则由 **Claude Code 强制执行，不是模型**。提示词或 `CLAUDE.md` 塑造 Claude *尝试*做什么，但不改变 Claude Code *允许*什么。）

## 五、怎么选：按场景挑模式

| 场景 | 推荐模式 | 官方理由 |
|---|---|---|
| 刚上手 / 敏感工作 | `default`（Manual） | 每个动作都过目 |
| 日常写码、愿意事后看 diff | `acceptEdits` | 自动放行目录内编辑，范围外仍弹窗 |
| 大改动前先摸清代码库 | `plan` | 只读探索、先出方案，批准后才动手 |
| 长任务、想少被打断 | `auto` | 分类器后台把关，但**不保证安全** |
| CI / 受限脚本 | `dontAsk` | 白名单外一律拒绝，会话永不等待输入 |
| 隔离容器 / VM | `bypassPermissions` | 跳过一切检查；**勿**在能损害宿主机的地方用 |

两个官方反复强调的坑：**auto ≠ 关闭安全**（只是把人工确认换成分类器审查）；**`defaultMode: "auto"` 放错文件会静默失效**（要放用户级设置）。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 获取，入口为官方文档索引 https://code.claude.com/docs/llms.txt）：

- **Choose a permission mode** — https://code.claude.com/docs/en/permission-modes
  （六种模式定义与对照表、Manual/`default` 命名、auto 模式分类器机制与默认拦截/放行清单、回退阈值、dontAsk / bypassPermissions 细则、受保护路径）
- **Configure auto mode** — https://code.claude.com/docs/en/auto-mode-config
  （`autoMode.environment` 可信基础设施配置、hard_deny/soft_deny/allow 覆盖规则与 `"$defaults"` 陷阱、`claude auto-mode` 子命令）
- **Configure permissions** — https://code.claude.com/docs/en/permissions
  （分层权限体系、allow/ask/deny 定义与求值顺序、规则由 Claude Code 强制、managed settings 与企业管控）

> 相关文档（同主题详细版）：`Agent/ClaudeCode/Claude Code 权限模式与 auto mode 官方解读.md`、`Agent/ClaudeCode/Claude Code 权限模式与 auto mode 官方怎么说.md`、`Agent/ClaudeCode/Claude Code 权限模式（permission modes）官方说明.md`。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
