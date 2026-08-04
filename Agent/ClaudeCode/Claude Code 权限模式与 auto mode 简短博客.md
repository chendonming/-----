# Claude Code 权限模式（permission modes）与 auto mode：官方简短博客

> **一句话总结**：Claude Code 官方把权限控制设计成"**模式定基线 + 规则做微调**"两层。共 6 种模式：`default`（界面叫 Manual，默认逐项确认）、`acceptEdits`（自动放行文件编辑）、`plan`（只读探索、批准后再动手）、`auto`（由一个独立**分类器模型**做后台安全检查、基本不弹窗）、`dontAsk`（只跑预批准工具，面向 CI）、`bypassPermissions`（跳过一切检查，仅限隔离环境）。官方反复强调：**auto mode 减少弹窗但不保证安全**，`bypassPermissions` 对提示注入毫无防护、**只应在容器/VM 等隔离环境里用**。本文基于官方 `Choose a permission mode`、`Configure auto mode`、`Configure permissions` 三页整理，文末附参考来源。

Claude 想编辑文件、跑命令、发请求时会暂停请你批准。官方对权限模式的定位一句话：

> "Control whether Claude asks before editing files or running commands."
> （控制 Claude 在编辑文件或运行命令前**是否询问**。）

它管的是"那个暂停发生得多频繁"。核心心法是**两层叠加**：

> "Modes set the baseline. Layer permission rules on top to pre-approve or block specific tools. These controls apply in every mode, including `bypassPermissions`."
> （模式设定**基线**，再叠加权限规则来预放行或拦截特定工具；这些控制在**每个模式**下都生效。）

---

## 一、6 种模式一张表

| 模式 | 不问就执行 | 官方定位 |
|---|---|---|
| `default`（界面叫 **Manual**） | 只读 | 刚上手、敏感工作 |
| `acceptEdits` | 读、文件编辑、常见文件命令（`mkdir`/`touch`/`mv`/`cp` 等） | 事后看 diff 的编码迭代 |
| `plan` | 读，+ auto 可用时分类器放行的命令 | 改动前探索代码库 |
| `auto` | 一切，带后台安全检查 | 长任务、减少弹窗疲劳 |
| `dontAsk` | 只放行预批准工具 | 锁死的 CI 和脚本 |
| `bypassPermissions` | 一切 | **仅限**隔离的容器/VM |

切换方式三选一：CLI 里 `Shift+Tab` 循环；启动时 `claude --permission-mode plan`；或 settings 里写 `"permissions": { "defaultMode": "acceptEdits" }` 设为持久默认。

## 二、auto mode：用"分类器"换掉弹窗

> "Auto mode lets Claude execute without routine permission prompts. A separate classifier model reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read."
> （auto mode 让 Claude **无需常规权限弹窗**即可执行；一个**独立的分类器模型**在动作执行前审查，拦截超出你请求范围、指向未识别基础设施、或被恶意内容驱动的操作。）

**它不是"不检查"，是把"问你"换成"分类器把关"**。官方配套警告很清醒：

> "Auto mode reduces permission prompts but does not guarantee safety."
> （auto mode 减少权限弹窗，但**不保证安全**。）

- **默认拦**：`curl | bash`、外发敏感数据、生产部署/迁移、`git reset --hard`/`git clean -fd`、force push、`terraform destroy` 等。
- **默认放行**：工作目录内文件操作、按 lockfile 装依赖、读 `.env` 发给对应 API、只读 HTTP、往当前仓库 push。
- **可配置**：`autoMode.environment` 用**自然语言**告诉分类器哪些是可信基础设施——官方说对大多数组织这是**唯一要配的字段**；`allow`/`soft_deny`/`hard_deny` 可覆盖内置规则，⚠️ 不带 `"$defaults"` 会**整表替换**内置默认规则。
- **兜底**：分类器**连续拦 3 次或累计 20 次**，auto mode 自动暂停、恢复逐条询问（阈值不可配）。
- **前提**：所有套餐可用；模型在 Anthropic API 上需 Opus 4.6+ / Sonnet 4.6+ / Fable 5（Bedrock / Vertex / Foundry 上更严格：仅 Sonnet 5 / Opus 4.7+ / Fable 5）。若提示不可用，"是某条要求没满足，**不是临时故障**"。
- **防自我授权**：`defaultMode: "auto"` 写在项目设置里会被忽略，必须放 `~/.claude/settings.json`（仓库无法给自己开 auto mode）。

## 三、规则层：allow / ask / deny

模式之上还有一层工具级规则：

> "**Allow** rules let Claude Code use the specified tool without manual approval. **Ask** rules prompt for confirmation… **Deny** rules prevent Claude Code from using the specified tool."
> （Allow 免审批；Ask 每次强制确认；Deny 直接禁止。）

三个关键口径：

- **求值顺序固定**：先 `deny` → 再 `ask` → 后 `allow`，第一个命中决定结果；deny 无法被 allow 开例外。
- **规则由 Claude Code 强制，不是模型**：
  > "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."
  > （提示词/CLAUDE.md 只能塑造 Claude *想做什么*，不能改变 Claude Code *允许什么*。）
- **设置优先级**：托管（managed，不可覆盖）> 命令行 > 项目本地 > 项目共享 > 用户；任一层 deny 了，其他层都不能放行。

## 四、落地选型

| 场景 | 推荐模式 |
|---|---|
| 敏感工作 / 刚上手 | `default` |
| 边写边改、事后 git diff 审查 | `acceptEdits` |
| 大改动前调研 | `plan` |
| 长任务、信得过方向 | `auto` |
| CI / 无人值守 | `dontAsk` |
| 隔离容器/VM 里自主运行 | `bypassPermissions` |

最后记住官方给 `bypassPermissions` 的那句劝告——想要"少弹窗 + 后台安全检查"，优先选 auto mode：

> "`bypassPermissions` offers no protection against prompt injection or unintended actions. For background safety checks with far fewer permission prompts, use auto mode instead."

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 获取，入口为官方文档索引 https://code.claude.com/docs/llms.txt）：

- **Choose a permission mode** — https://code.claude.com/docs/en/permission-modes
  （6 种模式定义与对照表、Shift+Tab 切换与 `defaultMode`、auto mode 机制与默认拦/放清单、dontAsk/bypassPermissions、受保护路径）
- **Configure auto mode** — https://code.claude.com/docs/en/auto-mode-config
  （`autoMode.environment` 信任基础设施、`hard_deny`/`soft_deny`/`allow` 优先级与 `"$defaults"` 陷阱、`claude auto-mode` 子命令）
- **Configure permissions** — https://code.claude.com/docs/en/permissions
  （allow/ask/deny 定义与求值顺序、分层权限系统、规则由 Claude Code 强制、设置优先级）

> 相关文档：`Agent/ClaudeCode/Claude Code 权限模式与 auto mode 官方解读.md`（完整版）、`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`。web 抓取环境配置见 `Agent/ClaudeCode/让web_fetch生效.md`。
