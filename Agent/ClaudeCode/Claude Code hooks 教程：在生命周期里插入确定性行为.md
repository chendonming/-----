# Claude Code hooks 教程：在生命周期里插入确定性行为

> **一句话总结**：**hooks 是你在 Claude Code 生命周期特定时刻自动执行的"确定性"动作——编辑文件后自动格式化、执行危险命令前拦截、Claude 等你输入时发桌面通知、上下文压缩后重新注入关键约定。** 它区别于让 LLM"自己决定要不要做"：只要事件触发、matcher 命中，shell 命令就一定跑。hooks 有五种形态（command / http / mcp_tool / prompt / agent），通过 **stdin 收 JSON、exit code + stdout 回决策**，并可与权限系统协作实现"用户无法绕过的策略"。
>
> 本文基于 Claude Code 官方文档 `Automate actions with hooks`（hooks 指南）与 `Hooks reference`（hooks 参考）整理，关键事件、配置 schema、示例均为官方原文，文末附参考来源。

很多人第一次接触 Claude Code 的 hooks，是看到某个项目里"每次写完代码文件它都会自动跑一遍 Prettier"或者"想 `rm -rf` 的时候被当场拦下"。这两个效果看起来很"魔法"，其实底层都是同一个机制：**hooks**。

它的核心价值一句话就能讲清：**确定性（deterministic）**。

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you deterministic control: certain actions always happen rather than relying on the LLM to choose to run them."
> （hooks 是用户定义的 shell 命令，Claude Code 在生命周期特定节点运行它们——这给了你确定性控制：某些动作**一定会**发生，而不是依赖 LLM 自己选择要不要做。）

参考手册里对 hooks 的定义更完整一些，因为它不只 shell 命令：

> "Hooks are user-defined shell commands, HTTP endpoints, or LLM prompts that execute automatically at specific points in Claude Code's lifecycle."
> （hooks 是用户定义的 shell 命令、HTTP 端点或 LLM 提示词，在 Claude Code 生命周期特定节点自动执行。）

---

## 一、先搞懂：hooks 为什么存在，和 skills / MCP / subagent 有什么不同

hooks 是 Claude Code 的"硬性钩子"，另外几种扩展机制各有分工：

| 扩展机制 | 形态 | 是否"一定会执行" | 典型用途 |
|---|---|---|---|
| **CLAUDE.md** | 静态指令文本 | 随会话/目录加载 | 项目约定、规范，无需跑脚本 |
| **Skills** | 指令 + 可执行命令 | 由 Claude 或用户按需调用 | 让 Claude"会做"某类任务 |
| **Subagents** | 隔离上下文的任务代理 | 由 Claude 调用 | 把探索/评审放到独立上下文里 |
| **MCP** | 外部工具/数据源 | 由 Claude 调用工具 | 接内部服务、查询数据库 |
| **Hooks** | shell 命令 / HTTP / LLM 判断 | **事件触发即执行，不依赖 LLM 决定** | 格式化、拦截、通知、注入上下文、审计 |

一个直觉的理解：skills/subagents/MCP 是"让 Claude 多做点事"，hooks 是"在 Claude 干活的前后，**你**插一段无论如何都要跑的代码"。官方指南的原话点出了这个分工：

> "Use hooks to enforce project rules, automate repetitive tasks, and integrate Claude Code with your existing tools."
> （用 hooks 来强制项目规则、自动化重复任务、把 Claude Code 和你的现有工具集成。）

## 二、配置：三层嵌套结构，六个可写位置

### 1. 配置位置与作用域

hooks 写在 JSON settings 里。放在哪，作用域就覆盖到哪：

| 位置 | 作用域 | 能否随仓库分享 |
|---|---|---|
| `~/.claude/settings.json` | 你机器上的所有项目 | 否，仅本机 |
| `.claude/settings.json` | 单个项目 | **是**，可提交进仓库 |
| `.claude/settings.local.json` | 单个项目 | 否，会被 gitignore |
| 受管策略设置（managed policy settings） | 组织级 | 是，管理员控制 |
| 插件 `hooks/hooks.json` | 插件启用时 | 是，随插件分发 |
| skill / subagent 的 frontmatter | 该组件活跃时 | 是，定义在组件文件里 |

关键点：**不同 settings 层的 hooks 是"合并"而不是"覆盖"**——用户、项目、本地 settings 各自追加自己的 hooks，不会冲掉受管设置。项目级 hooks 写进 `.claude/settings.json` 后能随仓库提交，是团队复用规则的标准姿势。

### 2. 三层嵌套：事件 → matcher 组 → handler

一份 hooks 配置从外到内是三层：

```json
{
  "hooks": {
    "PostToolUse": [                  // 第 1 层：hook 事件（生命周期节点）
      {
        "matcher": "Edit|Write",      // 第 2 层：matcher 组（过滤何时触发）
        "hooks": [                    // 第 3 层：hook handler（真正执行的东西）
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

官方对三个层级的名词定义很严谨：

> 本页对每一层用特定术语：**hook event**（生命周期节点）、**matcher group**（过滤器）、**hook handler**（真正运行的 shell 命令、HTTP 端点、MCP 工具、prompt 或 agent）。单独说 "hook" 指整个特性。

验证配置的入口是 `/hooks`：它会列出所有 hook 事件，每个已配置的事件旁边显示数量，点进去能看到事件、matcher、类型、来源文件、完整命令。注意官方特别说明：

> "The `/hooks` menu is read-only. To add, modify, or remove hooks, edit your settings JSON directly or ask Claude to make the change."
> （`/hooks` 菜单是**只读**的。增删改 hooks 要直接编辑 settings JSON，或让 Claude 帮你改。）

## 三、事件生命周期：hooks 在哪些时刻触发

官方把事件分成三种节奏（cadence）：

> "Events fall into three cadences:
> - once per session: `SessionStart` and `SessionEnd`
> - once per turn: `UserPromptSubmit`, `Stop`, and `StopFailure`
> - on every tool call inside the agentic loop: `PreToolUse` and `PostToolUse`"

（事件分三种节奏：**每个会话一次**——`SessionStart` / `SessionEnd`；**每一轮一次**——`UserPromptSubmit` / `Stop` / `StopFailure`；**agentic loop 内每次工具调用**——`PreToolUse` / `PostToolUse`。）

除了这三类核心事件，还有一批专门事件。下面这张表挑最常用的列出来（完整 30+ 个事件见参考手册）：

| 事件 | 何时触发 | 能否阻止动作 |
|---|---|---|
| `SessionStart` | 会话开始或恢复 | 否（只能注入上下文/环境变量） |
| `UserPromptSubmit` | 你提交提示词、Claude 处理之前 | **是**（可挡掉提示词） |
| `PreToolUse` | 工具调用执行之前 | **是**（拦截/改写工具入参） |
| `PermissionRequest` | 即将向你请求权限时 | **是**（代你 allow/deny） |
| `PostToolUse` | 工具成功执行之后 | 否（工具已跑完，可注入反馈/改写输出） |
| `PostToolUseFailure` | 工具执行失败后 | 否（可注入失败上下文） |
| `Stop` | Claude 回答完毕 | **是**（挡着不让停，让它继续干） |
| `SubagentStart` / `SubagentStop` | subagent 启动/结束 | Stop 可阻止，Start 不可 |
| `Notification` | Claude Code 发通知时 | 否（转发通知到别处） |
| `ConfigChange` | 会话中配置文件被改动 | **是**（可阻止变更生效） |
| `CwdChanged` / `FileChanged` | 目录切换 / 被监听文件变化 | 否（配合 direnv 重载环境） |
| `PreCompact` / `PostCompact` | 上下文压缩前后 | Pre 可阻止压缩 |
| `SessionEnd` | 会话终止 | 否（清理/统计） |

## 四、通信协议：stdin 收 JSON，exit code + stdout 回决策

hooks 和 Claude Code 之间没有"魔法变量"，就靠标准输入输出：

- **输入**：事件触发时，Claude Code 把事件相关数据以 JSON 形式写到 hook 的 **stdin**（command hook）或 POST 请求体（HTTP hook）。
- **输出**：hook 通过 **exit code** 告诉 Claude Code 接下来怎么办，用 **stdout/stderr** 传内容。

### 1. 输入：每个事件带不同的 JSON 字段

所有事件都带公共字段（`session_id`、`cwd`、`hook_event_name` 等），事件各自再追加专属字段。比如一个 `npm test` 命令触发的 `PreToolUse` hook，stdin 收到的是：

```json
{
  "session_id": "abc123",
  "cwd": "/Users/sarah/myproject",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

脚本里用 `cat` 读 stdin，再用 `jq` 取值即可。

### 2. 输出：exit code 是主信号

这是最容易踩坑的地方，官方用加粗警告强调了它：

> "For most hook events, only exit code 2 blocks the action. Claude Code treats exit code 1 as a non-blocking error and proceeds with the action, even though 1 is the conventional Unix failure code. If your hook is meant to enforce a policy, use `exit 2`."
> （对大多数事件，**只有 exit code 2 能阻止动作**。Claude Code 把 exit code 1 当作非阻塞错误、照常继续，尽管 1 是 Unix 惯例上的"失败码"。如果你的 hook 是要强制策略，**用 exit 2**。）

| exit code | 含义 | 效果 |
|---|---|---|
| **0** | 无异议 | 动作照常走（对 `PreToolUse` 来说并不等于"批准"，正常权限流程仍会走）；`UserPromptSubmit`/`SessionStart` 等事件还会把 stdout 作为上下文喂给 Claude |
| **2** | 阻止 | 阻止动作；把 stderr 作为反馈回传给 Claude，让它调整方案 |
| **其他**（含 1） | 非阻塞错误 | 动作继续；对话里出现 `<hook name> hook error` 提示，仅 stderr 首行以 `Failed with non-blocking status code:` 前缀展示 |

一个拦截危险命令的最小脚本：

```bash
#!/bin/bash
# 从 stdin 读 JSON，检查命令
input=$(cat)
command=$(jq -r '.tool_input.command' <<<"$input")

if [[ "$command" == rm* ]]; then
  echo "Blocked: rm commands are not allowed" >&2   # stderr 成为 Claude 的反馈
  exit 2                                            # exit 2 = 阻止动作
fi

exit 0  # 无决策：正常权限流程继续
```

### 3. 进阶输出：exit 0 + stdout 打 JSON

exit code 只能表达"放行/阻止"二选一。要更精细的控制（改写工具入参、注入上下文、决定 allow/deny/ask），exit 0 并在 stdout 输出一个 JSON 对象。官方特别提醒：**两种方式二选一，别混用**——exit 2 时任何 JSON 都会被忽略。

以 `PreToolUse` 为例，拒绝一次工具调用并告诉 Claude 原因：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

最常用的两个输出字段：

- **`additionalContext`**：向 Claude 上下文注入一段字符串，Claude Code 会把它包成 system reminder 插到 hook 触发的那个位置：

  > "The `additionalContext` field passes a string from your hook into Claude's context window. Claude Code wraps the string in a system reminder and inserts it into the conversation at the point where the hook fired."
  > （`additionalContext` 把一段字符串传入 Claude 的上下文窗口。Claude Code 把它包成一条 system reminder，插入到 hook 触发位置。）

- **`decision` + `reason`**：顶层 `decision: "block"` 用于 `PostToolUse`、`Stop`、`UserPromptSubmit` 等事件，阻止动作并给出理由。

不同事件的决策字段各不相同，官方给了一张速查表（节选）：

| 事件 | 决策模式 | 关键字段 |
|---|---|---|
| `PreToolUse` | `hookSpecificOutput` | `permissionDecision`（allow/deny/ask/defer）、`permissionDecisionReason` |
| `PermissionRequest` | `hookSpecificOutput` | `decision.behavior`（allow/deny） |
| `PostToolUse` / `Stop` 等 | 顶层 `decision` | `decision: "block"`、`reason` |
| `UserPromptSubmit` | 顶层 `decision` + `hookSpecificOutput` | `decision: "block"` 挡提示词、`additionalContext` 注入上下文 |
| `WorktreeCreate` | 返回路径 | command hook 在 stdout 打印路径 |

## 五、matcher 与 `if`：让 hook 只在"该跑的时候"跑

没有 matcher，hook 会在事件每次发生时都执行。`matcher` 在"组"这一层做第一次过滤，`if` 在"handler"这一层做第二次过滤。

### 1. matcher 按事件类型过滤不同字段

比如工具类事件按 **tool name** 过滤，`SessionStart` 按**会话如何开始**过滤，`Notification` 按**通知类型**过滤。常见例子：

| 事件 | matcher 过滤什么 | 示例值 |
|---|---|---|
| `PreToolUse`/`PostToolUse`/`PermissionRequest` 等 | 工具名 | `Bash`、`Edit\|Write`、`mcp__.*` |
| `SessionStart` | 会话怎么开始的 | `startup`、`resume`、`clear`、`compact`、`fork` |
| `Notification` | 通知类型 | `permission_prompt`、`idle_prompt`、`auth_success` |
| `SubagentStart`/`Stop` | agent 类型 | `general-purpose`、`Explore`、`Plan` |

matcher 的求值规则也值得记一下：

| matcher 值 | 求值方式 | 例子 |
|---|---|---|
| `"*"`、`""` 或缺省 | 匹配一切 | 每次事件都触发 |
| 仅含字母/数字/`_`/`-`/空格/`,`/`\|` | **精确字符串列表**（`\|` 或 `,` 分隔） | `Edit\|Write` 只匹配 Edit 和 Write 两个工具 |
| 含其他任何字符 | **JS 正则**（非锚定） | `mcp__memory__.*` 匹配 memory server 的所有工具 |

> 工具类事件还可以用 `if` 字段按"工具名 + 参数"一起过滤。它复用权限规则的语法，例如 `"Bash(git *)"` 只在 Claude 执行 git 命令时跑，`"Edit(*.ts)"` 只处理 TypeScript 文件。

### 2. 多个 hook 同时命中时：都跑，取最严

同事件的多个 hook 是**并行执行**的，一个 hook 返回 deny 不会阻止其他 hook 跑完。之后 Claude Code 合并结果，官方原话：

> "For `PreToolUse` permission decisions, the most restrictive answer applies, in the order `deny`, `defer`, `ask`, `allow`."
> （对 `PreToolUse` 的权限决策，取**最严格**的答案，优先级为 deny > defer > ask > allow。）

> "When multiple hooks match the same event, every hook's command runs to completion before Claude Code merges the results. One hook returning `deny` doesn't stop sibling hooks from executing."
> （同事件命中多个 hook 时，**每个 hook 都会跑完**才合并结果。一个 hook 返回 deny 不会阻止其他 hook 执行。）

所以"日志 hook"和"拦截 hook"可以共存：日志 hook 把命令写进文件、exit 0，拦截 hook 对 `rm -rf` exit 2——拦截生效，日志也照写。**别依赖某个 hook 的 deny 去抑制另一个 hook 的副作用。**

## 六、五种 handler 类型：command / http / mcp_tool / prompt / agent

每个 handler 有一个 `type` 字段。绝大多数场景用 `command`（跑 shell 命令），官方为"需要判断力"的场景提供了 LLM 型 hook：

| 类型 | 跑什么 | 返回方式 | 典型用途 | 备注 |
|---|---|---|---|---|
| **`command`** | shell 命令 | exit code + stdout/stderr | 格式化、拦截、通知、审计 | 最常用，默认超时 10 分钟 |
| **`http`** | POST 事件 JSON 到 URL | HTTP 响应体 | 团队共享审计服务、云函数 | 不能靠 HTTP 状态码阻止动作，要 2xx + JSON body |
| **`mcp_tool`** | 调用已连接的 MCP server 上的工具 | 工具文本输出 | 让已有 MCP 服务处理 hook 逻辑 | server 必须已连接，不触发 OAuth |
| **`prompt`** | 发给 Claude 模型做单轮判断 | `{"ok": true/false, "reason": ...}` | "检查任务是否都完成了"这类需要判断的 | 默认 Haiku，返回 yes/no 决策 |
| **`agent`** | 拉起一个能读文件/搜代码的 subagent | 同 prompt 格式 | "验证测试真的都过了" | **实验性**，官方建议生产环境优先用 command |

### prompt hook：把"要不要停"交给模型

有些决策不是"命令匹配"能解决的，需要理解语义。`prompt` hook 的用法：

> "For decisions that require judgment rather than deterministic rules, use `type: "prompt"` hooks. Instead of running a shell command, Claude Code sends your prompt and the hook's input data to a Claude model, Haiku by default, to make the decision."
> （对需要判断力而非确定性规则的决策，用 `type: "prompt"` hook。Claude Code 把你的 prompt 和 hook 的输入数据发给 Claude 模型（默认 Haiku）来做决策。）

一个"检查任务是否完成，没完成就继续干"的 `Stop` hook：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if all tasks are complete. If not, respond with {\"ok\": false, \"reason\": \"what remains to be done\"}."
          }
        ]
      }
    ]
  }
}
```

模型返回 `"ok": false` 时，`reason` 会被回喂给 Claude 作为下一条指令，让它继续工作。

### agent hook：需要"查证"时用

当判断需要看实际文件、跑测试时，`prompt` hook 的信息不够，用 `agent` hook：

> "Unlike prompt hooks, which make a single LLM call, agent hooks spawn a subagent that can read files, search code, and use other tools to verify conditions before returning a decision."
> （和只做单次 LLM 调用的 prompt hook 不同，agent hook 会拉起一个 subagent，它**能读文件、搜代码、用其他工具**去核实条件再返回决策。）

官方对两者选型给了一句口诀：hook 输入数据本身够判断 → 用 prompt；需要对着真实代码库验证 → 用 agent。但注意它的实验性警告：

> "Agent hooks are experimental. Behavior and configuration may change in future releases. For production workflows, prefer command hooks."
> （agent hooks 是**实验性**的，行为与配置可能在未来版本变化。生产工作流优先用 command hooks。）

## 七、动手实践：六个官方示例

以下配置块都来自官方指南，可直接复制到 settings 文件里。

### 1. Claude 需要你时发桌面通知（`Notification`）

`Notification` 事件在 Claude 等你输入或要权限时触发。Windows（PowerShell）版：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude Code needs your attention', 'Claude Code')\""
          }
        ]
      }
    ]
  }
}
```

`matcher` 留空对所有通知类型生效。想只对某类通知触发，可设 `permission_prompt`、`idle_prompt`、`auth_success` 等。

### 2. 编辑后自动格式化（`PostToolUse` + `Edit|Write`）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

原理：hook 从 stdin 的 JSON 里用 `jq` 取出被编辑的文件路径，交给 Prettier。注意：Claude 也可能通过 `Bash` 工具改文件——如果 hook 要覆盖所有文件变更（合规审计场景），官方建议再加一个 `Stop` hook 每轮扫描工作树，或让脚本用 `git status --porcelain` 列出变更文件。

### 3. 阻止编辑受保护文件（`PreToolUse` + exit 2）

官方用了一个独立脚本 `.claude/hooks/protect-files.sh`，命中 `.env` / `package-lock.json` / `.git/` 就 exit 2：

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED_PATTERNS=(".env" "package-lock.json" ".git/")

for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2
  fi
done

exit 0
```

配置（`.claude/settings.json`）里用 `${CLAUDE_PROJECT_DIR}` 引用项目内脚本，`PreToolUse` 在 `Edit`/`Write` 执行前拦截：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ]
  }
}
```

Claude 会收到 `Blocked:` 的 stderr 作为反馈，从而调整方案。

### 4. 上下文压缩后重新注入关键约定（`SessionStart` + `compact`）

上下文压缩会把对话摘要化，可能丢掉重要细节。用 `compact` matcher 在每次压缩后重注入约定：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Reminder: use Bun, not npm. Run bun test before committing. Current sprint: auth refactor.'"
          }
        ]
      }
    ]
  }
}
```

任何命令写到 stdout 的文本都会被加进 Claude 的上下文；换成 `git log --oneline -5` 就能展示最近提交。静态约定优先用 CLAUDE.md（不需要跑脚本）。

### 5. 审计配置变更（`ConfigChange`）

外部进程/编辑器改动配置文件时，`ConfigChange` 触发，把每次变更追加进审计日志：

```json
{
  "hooks": {
    "ConfigChange": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "jq -c '{timestamp: now | todate, source: .source, file: .file_path}' >> ~/claude-config-audit.log"
          }
        ]
      }
    ]
  }
}
```

matcher 可按配置来源过滤：`user_settings`、`project_settings`、`local_settings`、`policy_settings`、`skills`。exit 2 或返回 `{"decision": "block"}` 可阻止变更生效（`policy_settings` 例外，不可被阻止）。

### 6. 自动批准特定权限请求（`PermissionRequest` + `ExitPlanMode`）

自动放行 `ExitPlanMode`（Claude 展示完计划请求继续时），省去每次弹窗。这需要 hook 向 stdout 写 JSON 决策：

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\": {\"hookEventName\": \"PermissionRequest\", \"decision\": {\"behavior\": \"allow\"}}}'"
          }
        ]
      }
    ]
  }
}
```

官方专门提醒：**matcher 越窄越好**。匹配 `.*` 或留空会批准所有权限请求，包括文件写入和 shell 命令。

## 八、hooks 与权限系统：只能收紧，不能放松

这是 hooks 最容易被误用的一点。`PreToolUse` 在**任何**权限模式下都会先于权限检查触发：

> "PreToolUse hooks fire before any permission-mode check, in every permission mode, including `dontAsk`. A hook that returns `permissionDecision: "deny"` blocks the tool even in `bypassPermissions` mode or with `--dangerously-skip-permissions`. This lets you enforce policy that users can't bypass by changing their permission mode."
> （`PreToolUse` 在任何权限模式下都先于权限检查触发，包括 `dontAsk`。返回 `deny` 的 hook 即使在 `bypassPermissions` 模式或 `--dangerously-skip-permissions` 下也能拦住工具。这让你能强制用户**无法通过改权限模式绕过**的策略。）

但反过来不成立：

> "The reverse is not true: a hook returning `"allow"` doesn't bypass deny rules from settings... Hooks can tighten restrictions but not loosen them past what permission rules allow."
> （反向不成立：hook 返回 `"allow"` 不会绕过 settings 里的 deny 规则。**hooks 能收紧限制，但不能放松到权限规则允许的范围之外。**）

一句话记忆：**hook 能"多拦"，不能"放水"。**

## 九、高级能力：async 后台执行、defer 延迟恢复、CLAUDE_ENV_FILE

### 1. async：长任务丢到后台

默认 hooks 会阻塞 Claude 直到跑完。对部署、测试套件这类长任务，加 `"async": true` 让它后台跑、Claude 继续干活：

> "By default, hooks block Claude's execution until they complete. For long-running tasks like deployments, test suites, or external API calls, set `"async": true` to run the hook in the background while Claude continues working."
> （默认 hooks 会阻塞 Claude 直到完成。对部署、测试套件、外部 API 调用这类长任务，设 `"async": true` 让 hook 后台运行、Claude 继续工作。）

代价：async hooks **不能**阻止或控制动作（决策字段全部无效），输出在下一轮对话才送达。`async` 只对 `command` 类型有效。

### 2. CLAUDE_ENV_FILE：把环境变量持久化给后续 Bash 命令

`SessionStart`、`Setup`、`CwdChanged`、`FileChanged` 这类 hook 有 `CLAUDE_ENV_FILE` 环境变量，指向一个文件路径。hook 往里面写 `export` 语句，之后 Claude 每次跑 Bash 都会以它为前置脚本加载——这就是官方给出的 direnv 集成姿势：`SessionStart` 启动时导出环境，`CwdChanged` 每次 `cd` 后重新导出。

### 3. defer：把"问用户"推迟给外部程序

`"defer"` 是 `PreToolUse` 的第四个决策值，专给 `claude -p` 子进程集成（Agent SDK 应用、自定义 UI）用：hook 返回 defer，进程带着 `stop_reason: "tool_deferred"` 退出并保留待调用的工具；外部程序用自己的界面收集输入后，`claude -p --resume` 恢复，hook 再返回 allow + `updatedInput` 让工具执行。**只在 `-p` 非交互模式生效**，交互会话里会被忽略。

## 十、安全考量：hooks 是"全权限"代码

这是必须牢记的红线。command hooks 以你的系统用户完整权限执行 shell：

> "Command hooks execute shell commands with your full user permissions. They can modify, delete, or access any files your user account can access. Review and test all hook commands before adding them to your configuration."
> （command hooks 以你的**完整用户权限**执行 shell 命令，能改、能删、能访问你账号能访问的任何文件。加进配置前务必审查和测试每一条 hook 命令。）

官方给的安全实践清单：

- **校验并清理输入**：永远不要盲信输入数据
- **shell 变量总是加引号**：用 `"$VAR"` 而不是 `$VAR`
- **阻止路径穿越**：检查文件路径里的 `..`
- **用绝对路径**：脚本用 `${CLAUDE_PROJECT_DIR}` 引用；exec 形式无需引号
- **跳过敏感文件**：避开 `.env`、`.git/`、密钥等

另外，command hooks 无法触发 `/` 命令或工具调用，`additionalContext` 注入的内容也只是 Claude 当纯文本读的 system reminder——不要指望 hooks 里再嵌套一轮"让 Claude 做事"。

## 十一、调试技巧

- **`/hooks`**：确认 hook 注册在正确的事件下、查看来源与完整命令。
- **`claude --debug-file /tmp/claude.log`**：把完整执行细节（哪些 hook 命中、exit code、stdout/stderr）写进指定日志，另一个终端 `tail -f` 实时看。
- **`Ctrl+O`**：打开 transcript 视图，检查 hook 运行结果——成功（exit 0）不显示任何东西；exit 2 显示 stderr；其他 exit code 显示 `hook error` 提示。
- **手动测脚本**：`echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./my-hook.sh`，再用 `echo $?` 看退出码。
- **`/hooks` 显示空**：等几秒让文件 watcher 生效，或重启会话强制重载；确认 JSON 合法（不允许尾逗号和注释）。
- **hook 没跑**：确认 matcher 精确匹配（区分大小写）、触发的是正确事件（PreToolUse 在工具前、PostToolUse 在工具后）。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 抓取官方文档原文核对）：

- **Automate actions with hooks（hooks 指南）** — https://code.claude.com/docs/en/hooks-guide
  （hooks 定义与"确定性控制"、`/hooks` 菜单、全部常用事件表、matcher 过滤、五种 handler 类型、配置位置与作用域、prompt/agent/HTTP hooks、六个可复制的实战示例、限制与故障排查、调试技巧）
- **Hooks reference（hooks 参考手册）** — https://code.claude.com/docs/en/hooks
  （事件生命周期三种节奏、三层配置结构、matcher 求值规则、`if` 字段、JSON 输入/输出 schema、exit code 2 逐事件行为表、决策控制速查表、五个 hook 类型字段、async 后台执行、defer 延迟恢复、安全考量、Windows PowerShell、debug 日志）
- **官方文档索引** — https://code.claude.com/docs/llms.txt
  （用于定位全部 hooks 相关页面；确认 hooks 指南与参考是唯一两个 hooks 核心专页）

文内 bash/JSON 示例中的脚本（`protect-files.sh`、`block-rm.sh`、`run-tests-async.sh`、`plain-display.sh` 等）与官方给出的完整参考实现一致，另可参考官方仓库的 Bash command validator 示例：https://github.com/anthropics/claude-code/blob/main/examples/hooks/bash_command_validator_example.py

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（CLAUDE.md 与上下文管理，与 hooks 互补）、`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`（subagent/MCP 扩展机制对比）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
