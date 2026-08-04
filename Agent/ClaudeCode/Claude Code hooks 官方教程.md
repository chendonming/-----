# Claude Code hooks 官方教程：在生命周期节点自动执行命令

> **一句话总结**：**Hooks（钩子）是你在 Claude Code 生命周期关键节点上自动执行的命令/脚本/HTTP 端点/LLM 提示，官方文档给它的定位是"确定性控制"——某些动作"一定发生"，而不是依赖 LLM 自己决定要不要做。** 它可以用来：编辑后自动格式化、拦截危险命令、桌面通知、压缩后重新注入上下文、自动审批特定权限等。配置入口是 settings.json 里的 `hooks` 块（事件 → matcher 过滤 → handler 三层嵌套），通过 stdin 收 JSON、用退出码和 stdout JSON 跟 Claude Code 通信。
>
> 本文基于 Claude Code 官方文档 `Hooks guide`（Automate actions with hooks）与 `Hooks reference` 整理，所有代码示例均取自官方文档，文末附参考来源。

Hooks 是 Claude Code 三种主要扩展机制里"面向确定性自动化"的那一种。官方 guide 开头第一句就把它的定位说得很清楚：

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you deterministic control: certain actions always happen rather than relying on the LLM to choose to run them."
> （Hooks 是用户定义的 shell 命令。Claude Code 在生命周期特定节点运行它们，这给了你**确定性控制**：某些动作**一定发生**，而不是依赖 LLM 自己选择去执行。）

而 reference 页给出了更完整的定义——它不限于 shell 命令：

> "Hooks are user-defined shell commands, HTTP endpoints, or LLM prompts that execute automatically at specific points in Claude Code's lifecycle."
> （Hooks 是用户自定义的 shell 命令、HTTP 端点或 LLM 提示，它们在 Claude Code 生命周期的特定节点自动执行。）

如果你还没配置过任何 hook，建议从官方 guide（`hooks-guide`）上手；完整的事件 schema、JSON 输入输出格式、async/HTTP/MCP 等高级特性则看 reference（`hooks`）——reference 开头原话就是这么分工的：

> "If you're setting up hooks for the first time, start with the guide."
> （如果是第一次配置 hooks，从 guide 开始。）

---

## 一、先搞清楚：hooks 在什么时候触发

官方 reference 把全部 hook 事件按触发频率分成三类节奏：

- **每会话一次**：`SessionStart`（会话开始/恢复）、`SessionEnd`（会话结束）
- **每轮一次**：`UserPromptSubmit`（你提交提示词后、Claude 处理前）、`Stop`（Claude 回复完）、`StopFailure`（本轮因 API 错误结束）
- **agentic loop 里每次工具调用**：`PreToolUse`（工具执行前，可拦截）、`PostToolUse`（工具成功后）

reference 原话：

> "once per session: `SessionStart` and `SessionEnd`; once per turn: `UserPromptSubmit`, `Stop`, and `StopFailure`; on every tool call inside the agentic loop: `PreToolUse` and `PostToolUse`"
> （每会话一次：`SessionStart` 和 `SessionEnd`；每轮一次：`UserPromptSubmit`、`Stop`、`StopFailure`；agentic loop 内每次工具调用：`PreToolUse` 和 `PostToolUse`）

这是理解 hooks 的地图：**你选一个事件（什么时候触发），再决定这个事件要不要过滤（matcher）、以及触发后跑什么（handler）。**

### 常用事件速查表

| 事件 | 触发时机 | 能不能阻断？ |
|---|---|---|
| `SessionStart` | 会话开始或恢复 | 否（stdout 会作为上下文注入） |
| `UserPromptSubmit` | 提交提示词后、Claude 处理前 | 能，且 stdout 会注入上下文 |
| `PreToolUse` | 工具调用**前** | **能**——可用 `deny` 拦截工具调用 |
| `PermissionRequest` | 工具调用需要你批准时 | 能——可用 `allow` 替你回答 |
| `PostToolUse` | 工具调用成功后 | 否（工具已执行，不能撤销） |
| `PostToolUseFailure` | 工具调用失败后 | 否 |
| `Notification` | Claude Code 发通知（等你输入/要权限） | 否 |
| `Stop` | Claude 回复完一轮 | 能（可用 `block` 让 Claude 继续干） |
| `PreCompact` / `PostCompact` | 上下文压缩前后 | `PreCompact` 能，`PostCompact` 否 |
| `SessionEnd` | 会话终止 | 否 |

（完整事件表见 reference，共约 30 个事件，包括 `SubagentStart/Stop`、`CwdChanged`、`FileChanged`、`ConfigChange`、`WorktreeCreate/Remove`、`Elicitation` 等。）

### 和其他扩展机制的分工

官方 guide 在开头就给了"如何选择扩展方式"的指引——hooks 的定位是**确定性自动化**，跟另外三种互补：

| 机制 | 定位（官方原文意） | 典型用途 |
|---|---|---|
| **Hooks** | 生命周期节点上自动执行命令 | 格式化、拦截、通知、注入上下文 |
| Skills | 给 Claude 额外的指令与可执行命令 | 教 Claude"某类任务怎么做" |
| Subagents | 在隔离上下文中跑任务 | 长调研、不污染主上下文 |
| Plugins | 打包扩展跨项目分享 | 把 hook/skill/subagent/MCP 打包分发 |

---

## 二、配置结构：settings.json 里的 `hooks` 块

### 三层嵌套

reference 把配置结构总结成三步嵌套：

> "Choose a hook event to respond to, like `PreToolUse` or `Stop`; Add a **matcher group** to filter when it fires, like "only for the Bash tool"; Define one or more **hook handlers** to run when matched"
> （选一个要响应的事件，比如 `PreToolUse` 或 `Stop`；加一个 **matcher 组**过滤触发条件，比如"只针对 Bash 工具"；定义一个或多个 **hook handler** 在命中时运行。）

对应到 JSON：

```json
{
  "hooks": {
    "PreToolUse": [                 // ← 第一层：事件
      {
        "matcher": "Bash",          // ← 第二层：matcher 过滤
        "hooks": [                  // ← 第三层：handler 列表
          {
            "type": "command",      // handler 类型：command/http/mcp_tool/prompt/agent
            "command": "..."        // 该类型专属字段
          }
        ]
      }
    ]
  }
}
```

### 配在哪里？（作用域与是否可分享）

| 位置 | 作用域 | 可分享 |
|---|---|---|
| `~/.claude/settings.json` | 你的所有项目 | 否，仅本机 |
| `.claude/settings.json` | 单个项目 | 是，可提交进仓库 |
| `.claude/settings.local.json` | 单个项目 | 否，默认 gitignore |
| Managed policy settings | 组织级 | 是，管理员控制 |
| 插件 `hooks/hooks.json` | 插件启用时 | 是，随插件打包 |
| Skill / agent 的 frontmatter | 组件激活期间 | 是，定义在组件文件里 |

一个关键语义：**不同层级的 hooks 是"合并"而不是"覆盖"**。reference 原话：

> "Hook entries merge across settings levels rather than replacing each other: user, project, and local settings add their own hooks without removing managed ones."
> （hook 条目在 settings 各层级之间是**合并**的，而不是互相替换：user、project、local 设置各自添加自己的 hooks，不会移除 managed 的那份。）

临时全部禁用可以设置 `"disableAllHooks": true`——但注意：managed policy 里的 hooks 除非在 managed 层也设了这个开关，否则仍然会跑。

### 五种 handler 类型

| 类型 | 做什么 | 关键字段 |
|---|---|---|
| `command` | 跑一条 shell 命令，stdin 收 JSON | `command`（必填）、`args`、`async`、`shell` |
| `http` | POST JSON 到某个 URL | `url`（必填）、`headers`、`allowedEnvVars` |
| `mcp_tool` | 调用已连接 MCP server 上的工具 | `server`、`tool`、`input`（均必填） |
| `prompt` | 单轮 LLM 评估，返回 yes/no 决策 | `prompt`（必填）、`model` |
| `agent` | 起一个子代理，可用工具验证后再决策（**实验性**） | `prompt`、`timeout` |

### 你的第一个 hook：桌面通知

官方 guide 的入门示例，就是让你在 Claude 等你输入时收到桌面通知。macOS 版本：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

Linux 换 `notify-send 'Claude Code' 'Claude Code needs your attention'`；Windows（PowerShell）换：

```json
{
  "type": "command",
  "command": "powershell.exe -Command \"[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude Code needs your attention', 'Claude Code')\""
}
```

空 `matcher` 表示所有通知都触发。官方还给了更细的 matcher 值：`permission_prompt`（需要批准工具）、`idle_prompt`（Claude 干完在等你）、`auth_success`（认证完成）等。

配置好之后，在 CLI 里输入 `/hooks` 可以打开只读的 hooks 浏览器，核对每个事件下配了哪些 hook、来自哪个设置文件。官方 guide 提醒：`/hooks` 菜单是**只读**的，增删改要直接编辑 settings JSON，或让 Claude 帮你改。

---

## 三、matcher：怎么限定触发条件

### matcher 语法

reference 给的 matcher 求值规则：

| matcher 值 | 求值方式 |
|---|---|
| `"*"`、`""` 或省略 | 匹配所有 |
| 只含字母/数字/`_`/`-`/空格/`,`/`\|` | 精确字符串，或用 `\|`、`,` 分隔的精确串列表 |
| 含其他任何字符 | 当作 JavaScript 正则（不锚定） |

注意：`|` 和 `,` 在较新版本里等价。官方示例最常用的写法就是 `"Edit|Write"`——只匹配文件编辑类工具。

### 每个事件按什么字段过滤

matcher 不是对事件做模糊匹配，而是针对**特定字段**：`PreToolUse`/`PostToolUse` 等匹配**工具名**；`SessionStart` 匹配会话如何开始（`startup`/`resume`/`clear`/`compact`/`fork`）；`Notification` 匹配通知类型；`SessionEnd` 匹配结束原因（`clear`/`logout`/`other`）……

另外有一批事件**不支持 matcher**，永远全量触发。reference 原话：

> "`UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `MessageDisplay`, and `CwdChanged` don't support matchers and always fire on every occurrence."

### 匹配 MCP 工具

MCP 工具按 `mcp__<server>__<tool>` 命名，比如 `mcp__github__search_repositories`。匹配某个 server 的全部工具用正则 `mcp__github__.*`；跨 server 匹配写操作类用 `mcp__.*__write.*`。官方特别强调：**`.*` 是必须的**——裸的 `mcp__memory` 匹配不到任何工具。

### `if` 字段：工具名 + 参数一起过滤

matcher 只在组层面按工具名过滤；`if` 字段则用权限规则语法（如 `"Bash(git *)"`、`"Edit(*.ts)"`）按**工具名+参数**过滤，连 hook 进程都不 spawn。但官方有句重要提醒：

> "Because the `if` filter is best-effort, use the permission system rather than a hook to enforce a hard allow or deny."
> （因为 `if` 过滤是 best-effort 的，强制放行/禁止应该用权限系统，而不是 hook。）

`if` 只对工具类事件生效：`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied`。

---

## 四、输入与输出：stdin 收 JSON，退出码 + stdout JSON 决策

### 输入：事件数据以 JSON 从 stdin 进来

每个事件都会把 JSON 通过 stdin（command hook）或 POST body（http hook）传给你的脚本。公共字段包括 `session_id`、`prompt_id`、`cwd`、`permission_mode`、`hook_event_name` 等；事件专属字段则不同。比如一个 `npm test` 的 Bash 调用，`PreToolUse` hook 收到的输入长这样：

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

### 输出：退出码决定下一步

| 退出码 | 含义 | 备注 |
|---|---|---|
| `0` | 无异议，动作正常继续 | 对 `PreToolUse` 而言**不等于放行**，正常权限流程仍要走 |
| `2` | **阻断**动作 | stderr 会作为反馈喂回给 Claude；Claude Code 只在这个退出码下处理 JSON |
| 其他 | 非阻断错误 | 动作继续，transcript 显示 `<hook name> hook error` |

官方 reference 关于退出码有一条极其关键的说明，值得整段记住：

> "For most hook events, only exit code 2 blocks the action. Claude Code treats exit code 1 as a non-blocking error and proceeds with the action, even though 1 is the conventional Unix failure code. If your hook is meant to enforce a policy, use `exit 2`."
> （对大多数 hook 事件，**只有退出码 2 会阻断动作**。Claude Code 把退出码 1 当作非阻断错误、继续执行——尽管 1 是 Unix 惯例的失败码。如果你的 hook 是要强制某个策略，用 `exit 2`。唯一的例外是 `WorktreeCreate`，任何非零退出码都会中止创建。）

### 结构化 JSON 输出：比退出码更强的控制

退出码只能"阻断或沉默"。想要更细的控制，就 `exit 0` 并在 stdout 打印 JSON。比如一个 `PreToolUse` hook 既拦截工具又告诉 Claude 原因：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

`PreToolUse` 的 `permissionDecision` 有四个值：`allow`（跳过交互式批准，但 deny/ask 规则仍生效）、`deny`（取消调用并喂原因给 Claude）、`ask`（照常弹出批准框）、`defer`（非交互 `-p` 模式下保留调用待 SDK 处理）。

**先记住一条铁律**——退出码和 JSON 二选一，别混用。guide 原话：

> "Use exit 2 to block with a stderr message, or exit 0 with JSON for structured control. Don't mix them: Claude Code ignores JSON when you exit 2."
> （用 exit 2 + stderr 消息来阻断，或用 exit 0 + JSON 做结构化控制。别混用：exit 2 时 Claude Code 会忽略 JSON。）

### 关键语义：沉默不等于放行

对 `PreToolUse` 而言，hook 保持沉默（exit 0 且无 JSON）**不构成批准**，正常的权限流程照旧。reference 原话：

> "The hook can deny the call, but staying silent doesn't approve it."
> （hook 可以拒绝调用，但沉默并不等于批准它。）

### 多 hook 并行与结果合并

同一事件命中多个 handler 时，全部**并行跑完**再合并结果；完全相同的 handler 会自动去重。合并规则里最实用的一条是权限决策取"最严格"：

> "For `PreToolUse` permission decisions, the most restrictive answer applies, in the order `deny`, `defer`, `ask`, `allow`."
> （对 `PreToolUse` 权限决策，最严格的答案生效，优先级从高到低为 `deny`、`defer`、`ask`、`allow`。）

`additionalContext` 文本则每个 hook 的都保留、一起传给 Claude。

---

## 五、实战：六个官方用例，照着抄就能用

以下示例均取自官方 guide（`hooks-guide`），各自解决一个具体问题。

### 1. 编辑后自动格式化（PostToolUse + `Edit|Write` + Prettier）

Claude 每次编辑完文件，自动跑一遍 Prettier。放在项目根目录的 `.claude/settings.json`：

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

`jq` 负责从 stdin 的 JSON 里取出被编辑的文件路径。（官方 note：`jq` 可用 `brew install jq` / `apt-get install jq` 安装。）

### 2. 拦截危险命令（PreToolUse + `Bash` + exit 2）

用一个脚本挡住 `rm -rf`。先写脚本 `.claude/hooks/block-rm-rf.sh`：

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q "rm -rf"; then
  echo "Blocked: rm -rf is not allowed" >&2   # stderr 变成 Claude 的反馈
  exit 2                                       # exit 2 = 阻断该工具调用
fi

exit 0  # 不决策，走正常权限流程
```

再注册 hook：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-rm-rf.sh"
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PROJECT_DIR}` 是项目根目录占位符，官方推荐用它引用项目内脚本。

### 3. 保护受保护文件（PreToolUse + `Edit|Write`）

阻止 Claude 改 `.env`、`package-lock.json`、`.git/` 等敏感文件。脚本 `.claude/hooks/protect-files.sh`：

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

注册（注意 macOS/Linux 需要 `chmod +x` 让脚本可执行）：

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

### 4. 压缩后重新注入上下文（SessionStart + `compact` matcher）

上下文压缩会丢细节。用 `SessionStart` + `compact` matcher，每次压缩后把关键约定重新注入——任何写到 stdout 的文本都会加进 Claude 的上下文：

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

把 `echo` 换成 `git log --oneline -5` 之类的动态命令也行。

### 5. 自动审批特定权限（PermissionRequest + `ExitPlanMode` + `allow`）

每次 Claude 展示完计划、问"是否继续"时自动放行，不打扰你。这个要用 **JSON 决策**而不是退出码：

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

官方特意提醒：**matcher 越窄越好**——用 `.*` 或留空会替你批准所有权限提示（包括文件写入和 shell 命令）。transcript 里会出现 "Allowed by PermissionRequest hook" 字样。注意 hook 路径放行后总是保持当前会话，不会像对话框那样能清上下文另起会话。

### 6. 记录所有 Bash 命令（PostToolUse + `Bash`）

把每条执行过的 shell 命令追加进日志，方便审计：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command' >> ~/.claude/command-log.txt"
          }
        ]
      }
    ]
  }
}
```

> 顺带一提：如果你的目的是"看到每一次文件改动"，官方 guide 提醒 Bash 也能改文件，`Edit|Write` matcher 会漏掉它。合规/审计场景应加一个 `Stop` hook 每轮扫一次工作树（`git status --porcelain`）。

---

## 六、进阶：prompt hooks、agent hooks、HTTP hooks 与 async

### Prompt hooks：让模型替你判断

判定需要"判断力"而不是确定性规则时，用 `type: "prompt"`。Claude Code 把你的 prompt 连同 hook 输入发给一个 Claude 模型（默认快速模型），让它返回 `{"ok": true/false, "reason": ...}`。这个例子让模型检查所有任务是否完成，没完成就让 Claude 继续干：

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

### Agent hooks：让子代理去验证（实验性）

比 prompt hook 更进一步：`type: "agent"` 会 spawn 一个**子代理**，它可以用 Read/Grep/Glob 等工具实际检查代码库再决策。默认超时 60 秒、最多 50 次工具调用。官方用 Warning 明确标注：

> "Agent hooks are experimental. Behavior and configuration may change in future releases. For production workflows, prefer command hooks."
> （Agent hooks 是实验性的，行为和配置可能在后续版本变化。生产工作流优先用 command hooks。）

### HTTP hooks：POST 给 Web 服务

想复用团队已有的审计服务/云函数，用 `type: "http"`。事件数据以 JSON 作为 POST body，响应体按同样的 JSON 输出格式解析。注意一个关键区别：**HTTP 状态码本身不能阻断动作**——要阻断，得返回 2xx + 带决策字段的 JSON body。非 2xx / 连接失败 / 超时一律是非阻断错误。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:8080/hooks/tool-use",
            "headers": { "Authorization": "Bearer $MY_TOKEN" },
            "allowedEnvVars": ["MY_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

`allowedEnvVars` 指定哪些环境变量可以在 header 里被插值——`$MY_TOKEN` 只有列在这里才会被解析，其他 `$VAR` 会留空。

### Async hooks：后台跑、别阻塞

`"async": true` 让 hook 后台运行不阻塞主流程；`"asyncRewake": true` 更进一步——后台跑，**退出码 2 时唤醒 Claude** 让它响应。官方对 `asyncRewake` 的原话：

> "wakes Claude on exit code 2... so it can react to a long-running background failure."
> （在退出码 2 时唤醒 Claude……让它能对长时间运行的后台失败做出反应。）

---

## 七、安全注意事项（官方明确警告）

这是最容易忽略、也最不该忽略的一节。reference 有专门的 `Security considerations` 小节，第一句就是：

> "Command hooks run with your system user's full permissions."
> （Command hooks 以你系统用户的**完整权限**运行。）

配套的 Warning 说得更直白：

> "Command hooks execute shell commands with your full user permissions. They can modify, delete, or access any files your user account can access. Review and test all hook commands before adding them to your configuration."
> （Command hooks 以你的完整用户权限执行 shell 命令，能修改、删除、访问你的账号能碰到的任何文件。**加进配置前，先审查并测试所有 hook 命令。**）

官方列出的 hook 安全最佳实践：

- **校验并清洗输入**：永远不要盲目信任输入数据
- **shell 变量要加引号**：用 `"$VAR"` 而不是 `$VAR`
- **阻止路径穿越**：检查文件路径里有没有 `..`
- **用绝对路径**：exec 形式里用 `${CLAUDE_PROJECT_DIR}` 无需引号，shell 形式要加双引号
- **跳过敏感文件**：避开 `.env`、`.git/`、密钥等

另有三条和权限体系相关的硬规则：

1. **`PreToolUse` 先于任何权限模式检查触发**，包括 `dontAsk`；返回 `deny` 时，即使在 `bypassPermissions` 或 `--dangerously-skip-permissions` 下也能阻断。也就是说——**hook 用来收紧权限，用户改不了权限模式也绕不过去**。
2. **反过来不成立**：hook 返回 `allow` 不能绕过 settings 里的 deny 规则，也不能压掉 connector 工具/MCP 工具的 `ask` 提示。官方原话："Hooks can tighten restrictions but not loosen them past what permission rules allow."（hooks 只能收紧限制，不能放松到权限规则允许之外。）
3. **注入上下文要当心提示注入**。通过 `additionalContext` 注入的文本，官方建议写成事实陈述而非命令口吻：

> "Write the text as factual statements rather than imperative system instructions... Text framed as out-of-band system commands can trigger Claude's prompt-injection defenses."
> （把文本写成事实陈述，而不是命令式系统指令……以"带外系统命令"口吻写成的文本会触发 Claude 的提示注入防御。）

---

## 八、排障速查

| 症状 | 官方给的排查方向 |
|---|---|
| Hook 不触发 | `/hooks` 确认它出现在正确事件下；matcher 大小写敏感、要和工具名完全一致；分清 `PreToolUse`（前）/`PostToolUse`（后） |
| transcript 显示 `hook error` | 脚本非零退出。手动测：`echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' \| ./my-hook.sh` 再看 `echo $?`；`command not found` 就用绝对路径或 `${CLAUDE_PROJECT_DIR}` |
| `/hooks` 里啥也没有 | 等几秒让文件监听生效，或重启会话；JSON 不能有尾逗号/注释；确认文件位置（项目用 `.claude/settings.json`，全局用 `~/.claude/settings.json`） |
| Stop hook 反复阻断 | 连续阻断 8 次会被覆盖；脚本里解析 `stop_hook_active` 字段、为 `true` 时直接 `exit 0`；必要时用 `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` 调高上限 |
| JSON 解析失败 | shell 形式会经 `sh -c`/Git Bash，shell profile 里的无条件 `echo` 会污染 stdout；把 profile 的 echo 包进 `if [[ $- == *i* ]]` 交互判断里 |

调试手段：启动时加 `claude --debug-file /tmp/claude.log` 然后 `tail -f` 看完整日志；会话中途可 `/debug` 开启。脚本成功（exit 0）时 transcript 不显示任何东西，想看是否真跑了就检查它的效果（如文件被格式化），或开 debug 日志。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 web_fetch 获取；`Hooks reference` 另以 curl 直抓原始 markdown 逐字核对关键引语）：

- **Automate actions with hooks（Hooks guide，上手指南）** — https://code.claude.com/docs/en/hooks-guide
  （hooks 是什么、第一个 hook 配置、六个实战用例：桌面通知/格式化/拦截危险命令/保护文件/压缩后注入上下文/自动审批、matcher 与 `if` 字段、prompt/agent/HTTP hooks、限制与排障）
- **Hooks reference（完整参考）** — https://code.claude.com/docs/en/hooks
  （全部约 30 个事件与触发时机、配置三层嵌套与五种 handler 类型、matcher 语法与按事件过滤字段、输入输出 schema、退出码语义、JSON 决策字段、async/asyncRewake、MCP 工具命名、`/hooks` 菜单、Security considerations）
- **Documentation index** — https://code.claude.com/docs/llms.txt
  （用于定位 hooks 相关页面：`hooks-guide`、`hooks`、`agent-sdk/hooks` 三页）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md）、`Agent/ClaudeCode/Claude Code 权限模式与 auto mode 官方怎么说.md`（权限模式——与 hook 的 allow/deny 语义直接相关）、`Agent/ClaudeCode/Claude Code 官方建议怎么用 subagent.md`（subagent——hook 之外的另一条扩展路径）。
