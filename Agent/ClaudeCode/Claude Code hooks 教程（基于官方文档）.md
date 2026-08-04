# Claude Code hooks 教程（基于官方文档）

> **一句话总结**：**hooks 是 Claude Code 里把"某些动作一定发生"从"靠 LLM 想起来才做"变成"确定性自动执行"的机制。** 官方定义是：hooks 是用户自定义的 shell 命令（也可扩展为 HTTP 端点、MCP 工具调用、LLM 判断、子代理验证），Claude Code 在生命周期特定节点自动运行它们。用它做"每次编辑后自动格式化""`rm -rf` 之前拦截""Claude 需要输入时弹通知"这类**必须每次发生、不需要 Claude 思考**的事；而"让 Claude 每次记得遵守某条规则"这类**要求而不是保证**的事，官方明确说应该用 hook 而不是只写进 CLAUDE.md。配置一次，一次写进 `settings.json`，一个 `/hooks` 命令查看，一个 `exit 2` 完成拦截。
>
> 本文基于 Claude Code 官方文档 `Automate actions with hooks`（指南）、`Hooks reference`（参考）、`Security`、`Explore Claude Code features`、`Debug your configuration` 整理，文末附参考来源。文中所有配置示例与事件清单均直接取自上述官方页面。

Claude Code 默认是"LLM 决定做什么"：它可能记得格式化、可能不记得。当某个动作**必须每次发生**时——比如保护 `.env` 不被改、每次编辑后跑 prettier、CI 里一次性初始化——把希望寄托在模型的"记性"上是不靠谱的。官方在 hooks 指南里点破了这一点：

> "Hooks are user-defined shell commands. Claude Code runs them at specific points in its lifecycle, which gives you deterministic control: certain actions always happen rather than relying on the LLM to choose to run them."
> （hooks 是用户定义的 shell 命令。Claude Code 在生命周期的特定节点运行它们，这给了你**确定性的控制**：某些动作**总是会**发生，而不是依赖 LLM 决定要不要做。）

这就是 hooks 与 CLAUDE.md / skill 的根本区别：**CLAUDE.md 是"请它遵守"，hooks 是"强制它发生"**。

---

## 一、hooks 是什么：事件 → matcher → handler 三级结构

官方参考文档把 hooks 定义为：

> "Hooks are user-defined shell commands, HTTP endpoints, or LLM prompts that execute automatically at specific points in Claude Code's lifecycle."
> （hooks 是用户自定义的 shell 命令、HTTP 端点或 LLM 提示词，在 Claude Code 生命周期的特定节点自动执行。）

配置上是一个**三级嵌套**结构（官方用三个专有名词区分层级）：

1. **hook event（事件）**：选择响应哪个生命周期节点，如 `PreToolUse`、`Stop`；
2. **matcher group（匹配组）**：用 `matcher` 过滤"何时触发"，如"只在 Bash 工具调用时"；
3. **hook handler（处理器）**：真正执行的命令 / HTTP 请求 / MCP 工具 / 提示词 / 子代理。

一个最小的、给 `PreToolUse` 事件加"编辑文件前拦截"的配置长这样：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh" }
        ]
      }
    ]
  }
}
```

最外层的 key（`PreToolUse`）是**事件**，数组第一层是**匹配组**（`matcher` 控制组是否激活），最内层 `hooks` 数组里每个对象是一个**处理器**。

### 写在哪里：作用域决定"谁能用"

| 位置 | 作用域 | 是否可分享 |
|---|---|---|
| `~/.claude/settings.json` | 你所有的项目 | 否，只在本机 |
| `.claude/settings.json` | 单个项目 | 是，可提交进仓库 |
| `.claude/settings.local.json` | 单个项目 | 否，Claude Code 写入时会 gitignore |
| 托管策略设置（managed policy settings） | 整个组织 | 是，管理员控制 |
| 插件 `hooks/hooks.json` | 插件启用时 | 是，随插件分发 |
| skill / subagent 的 frontmatter | 组件激活期间 | 是，定义在组件文件里 |

注意：hooks 从 settings 文件、托管策略和插件里注册后，**子代理里同样生效**——子代理调用工具时 `PreToolUse`/`PostToolUse` 会触发同样的 hooks（输入里会带上 `agent_id`/`agent_type` 标识）。

### 查看与调试：`/hooks`

在 Claude Code 里敲 `/hooks` 打开**只读**的 hooks 浏览器：它会列出每个事件、每个事件下已配置的 hook 数量，选中某个 hook 还能看到它的事件、matcher、类型、来源文件和完整命令。官方明确说：

> "The `/hooks` menu is read-only. To add, modify, or remove hooks, edit your settings JSON directly or ask Claude to make the change."
> （`/hooks` 菜单是只读的。要增删改 hooks，直接编辑 settings JSON，或让 Claude 来改。）

编辑 settings 文件后，文件监听器一般会自动热加载，无需重启会话。

---

## 二、hooks 在生命周期里的位置：31 个事件

官方把事件分成三种节奏：

> "Events fall into three cadences: once per session: `SessionStart` and `SessionEnd`; once per turn: `UserPromptSubmit`, `Stop`, and `StopFailure`; on every tool call inside the agentic loop: `PreToolUse` and `PostToolUse`…"
> （事件有三种节奏：每会话一次：`SessionStart` 和 `SessionEnd`；每轮一次：`UserPromptSubmit`、`Stop`、`StopFailure`；agentic loop 内每次工具调用：`PreToolUse` 和 `PostToolUse`……）

全量事件表（摘自官方参考文档，按生命周期顺序排列）：

| 事件 | 触发时机 |
|---|---|
| `SessionStart` | 会话开始或恢复时 |
| `Setup` | 用 `--init-only`、或 `-p` 模式配合 `--init`/`--maintenance` 启动时（一次性准备，适合 CI） |
| `UserPromptSubmit` | 你提交提示词、Claude 处理之前 |
| `UserPromptExpansion` | 用户输入的命令展开成提示词时（可拦截展开） |
| `PreToolUse` | 工具调用执行前（**可拦截**） |
| `PermissionRequest` | 工具调用需要权限决策时 |
| `PermissionDenied` | 工具调用被 auto mode 分类器拒绝时（返回 `{retry: true}` 可让模型重试） |
| `PostToolUse` | 工具调用成功之后 |
| `PostToolUseFailure` | 工具调用失败之后 |
| `PostToolBatch` | 一批并行工具调用全部结束后、下一次模型调用前 |
| `Notification` | Claude Code 发送通知时 |
| `MessageDisplay` | 助手消息流式显示期间 |
| `SubagentStart` | 子代理被创建时 |
| `SubagentStop` | 子代理结束时 |
| `TaskCreated` | 通过 `TaskCreate` 创建任务时 |
| `TaskCompleted` | 任务被标记完成时 |
| `Stop` | Claude 结束回应时 |
| `StopFailure` | 回合因 API 错误结束时（输出和退出码被忽略） |
| `TeammateIdle` | agent team 队友即将进入空闲时 |
| `InstructionsLoaded` | CLAUDE.md 或 `.claude/rules/*.md` 被加载进上下文时 |
| `ConfigChange` | 会话中配置文件发生变化时 |
| `CwdChanged` | 工作目录变化（如 Claude 执行 `cd`）时 |
| `DirectoryAdded` | 会话中途 `/add-dir` 或 SDK `register_repo_root` 添加目录后 |
| `FileChanged` | 被监视的文件在磁盘上变化时 |
| `WorktreeCreate` | 创建 worktree 时（可替换默认 git 行为，支持非 git VCS） |
| `WorktreeRemove` | 移除 worktree 时 |
| `PreCompact` | 上下文压缩前 |
| `PostCompact` | 上下文压缩完成后 |
| `Elicitation` | MCP server 在工具调用中请求用户输入时 |
| `ElicitationResult` | 用户回应 MCP 请求后、回应发回 server 前 |
| `SessionEnd` | 会话终止时 |

每个事件能**干什么**不一样——有的能拦截（exit 2 阻止动作），有的只能做"副作用"（记日志、发通知）。这个差异很关键，官方给了一张按事件列的"exit 2 行为表"，下面第五节会讲到。

---

## 三、五种 hook 类型：从"跑命令"到"让 AI 判断"

每种事件下，处理器（handler）的 `type` 决定它怎么运行：

| 类型 | 干什么 | 关键点 |
|---|---|---|
| **`command`** | 跑一条 shell 命令 | 最常用；stdin 收 JSON、stdout/退出码回传结果 |
| **`http`** | POST 事件数据到指定 URL | 端点的响应体用与 command hook 相同的 JSON 格式 |
| **`mcp_tool`** | 调用已连接的 MCP server 上的工具 | server 必须已连接；文本输出按 command hook 的 stdout 处理 |
| **`prompt`** | 单轮 LLM 判断（默认 Haiku） | 模型只回 `{"ok": true/false, "reason": ...}` |
| **`agent`** | 多轮子代理验证（可读文件、搜代码） | **实验性，官方警告生产环境优先用 command hook** |

官方对"什么时候该用判断型 hook"是这样定位的：

> "For decisions that require judgment rather than deterministic rules, you can also use prompt-based hooks or agent-based hooks that use a Claude model to evaluate conditions."
> （对于需要判断力而非确定性规则的决定，你也可以用 prompt-based hooks 或 agent-based hooks，让 Claude 模型来评估条件。）

判断型 hook 的典型场景：`Stop` 事件上问模型"所有任务是否真的做完了？"——命令型 hook 很难写出这种规则，但一个 prompt 就够了：

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

而 `agent` 型更进一步：它 spawn 一个**能实际读文件、跑命令验证**的子代理（默认 60 秒、最多 50 个工具回合），适合"跑一遍测试再让 Claude 收工"这类必须核对真实状态的场景：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

官方给的选择准则一句话：

> "Use prompt hooks when the hook input data alone is enough to make a decision. Use agent hooks when you need to verify something against the actual state of the codebase."
> （当仅凭 hook 输入数据就足以做决定时用 prompt hook；当需要对照代码库的实际状态做验证时用 agent hook。）

HTTP 型 hook 适合"把 hook 逻辑放到 web server / 云函数上"的场景——比如团队共用的审计服务：

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

注意：HTTP header 里的 `$VAR` 只有在 `allowedEnvVars` 里列出的变量才会被插值；HTTP 状态码本身**不能**拦截动作，要拦截必须返回 2xx + 带决策字段的 JSON 响应体。

**不是每个事件都支持每种类型。** 官方给了明确的支持矩阵：13 个事件支持全部五种类型（`PreToolUse`、`PostToolUse`、`Stop`、`SubagentStop`、`UserPromptSubmit`、`UserPromptExpansion`、`PostToolBatch`、`PostToolUseFailure`、`PermissionDenied`、`PermissionRequest`、`TaskCreated`、`TaskCompleted`、`TeammateIdle`）；`ConfigChange`、`Notification`、`SessionEnd`、`PreCompact`、`PostCompact`、`FileChanged`、`CwdChanged`、`Elicitation`、`ElicitationResult` 等只支持 command/http/mcp_tool；`SessionStart` 和 `Setup` 只支持 command/mcp_tool。

---

## 四、matcher 与 if：精准控制"这个 hook 何时跑"

一个 hook 配置里 `matcher` 控制**匹配组**在什么条件下激活。官方给了三种求值方式：

| matcher 值 | 求值方式 | 示例 |
|---|---|---|
| `"*"`、`""`、或省略 | 全部匹配 | 该事件的每次发生都触发 |
| 只含字母、数字、`_`、`-`、空格、`,`、`\|` | 精确字符串 / 用 `\|` 或 `,` 分隔的精确字符串列表 | `Bash` 只匹配 Bash 工具；`Edit\|Write` 匹配两者 |
| 含其他字符 | **非锚定的 JavaScript 正则** | `^Notebook` 匹配名字以 Notebook 开头的工具；`mcp__memory__.*` 匹配 memory server 的全部工具 |

每个事件匹配的"字段"不同：工具类事件（`PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`PermissionRequest`/`PermissionDenied`）匹配**工具名**；`SessionStart` 匹配启动方式（`startup`/`resume`/`clear`/`compact`/`fork`）；`Notification` 匹配通知类型；`SubagentStart`/`SubagentStop` 匹配代理类型；`ConfigChange` 匹配配置来源。而 `UserPromptSubmit`、`Stop`、`PostToolBatch`、`CwdChanged` 等事件**不支持 matcher**，每次必然触发——给这些事件加 matcher 会被静默忽略。

MCP 工具走 `mcp__<server>__<tool>` 命名，匹配时要带 `.*`：`mcp__memory__.*` 匹配 memory server 的所有工具；`mcp__.*__write.*` 跨 server 匹配名字以 write 开头的工具。插件捆绑的 server 名更长，是 `mcp__plugin_<插件名>_<server名>__<tool>`。

### `if`：比 matcher 更细的过滤

`matcher` 在"组"这一层按工具名过滤；`if` 字段则用**权限规则的语法**按"工具名 + 参数"一起过滤，命中才 spawn 进程。比如只拦 git 命令：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(git *)",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-git-policy.sh"
          }
        ]
      }
    ]
  }
}
```

`if` 对 Bash 会检查子命令、`$()`、反引号里的命令（如 `npm test && git push` 会命中 `Bash(git *)`，`echo $(git log)` 也会命中）。但官方提醒：这个解析是 best-effort 的，**解析不了时会 fail-open（照常跑 hook）**，所以真正的硬性 allow/deny 应该用权限系统而不是 hook。`if` 只在工具类事件上生效，加到其他事件会导致该 hook 永不运行。

---

## 五、输入与输出协议：stdin JSON、退出码、结构化 JSON

这是写 hook 脚本最核心的部分。官方总结为：

> "Hooks communicate with Claude Code through stdin, stdout, stderr, and exit codes. When an event fires, Claude Code passes event-specific data as JSON to your script's stdin. Your script reads that data, does its work, and tells Claude Code what to do next via the exit code."
> （hooks 通过 stdin、stdout、stderr 和退出码与 Claude Code 通信。事件触发时，Claude Code 把事件特定的 JSON 数据通过 stdin 传给脚本；脚本读数据、干活，然后用退出码告诉 Claude Code 接下来怎么办。）

### 输入（stdin 上的 JSON）

所有事件都带公共字段：`session_id`、`transcript_path`、`cwd`、`permission_mode`、`hook_event_name` 等；工具类事件再加 `tool_name`、`tool_input`、`tool_use_id`。比如一个 `Bash("npm test")` 前的 `PreToolUse` hook 收到：

```json
{
  "session_id": "abc123",
  "cwd": "/Users/sarah/myproject",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" }
}
```

脚本里 `cat` 读 stdin，再用 `jq` 解析（官方示例都用 `jq`）。

### 退出码决定下一步

| 退出码 | 含义 |
|---|---|
| **0** | 无异议，动作正常进行。对 `PreToolUse` **不是**放行——正常权限流程仍会执行；对 `UserPromptSubmit`/`UserPromptExpansion`/`SessionStart`，stdout 内容会作为上下文给 Claude |
| **2** | **拦截**动作。把原因写进 stderr，Claude 会收到作为反馈从而调整做法 |
| **其他** | 非阻塞错误：动作继续，转录里出现 `<hook name> hook error` 提示 + stderr 第一行 |

一个经典的拦截脚本：

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q "drop table"; then
  echo "Blocked: dropping tables is not allowed" >&2   # stderr 变成 Claude 的反馈
  exit 2                                                # exit 2 = 拦截
fi

exit 0                                                  # 无决策，走正常流程
```

⚠️ 官方特别警告：**对大多数事件只有 exit 2 会拦截，exit 1 会被当作非阻塞错误放行**——即使 1 是 Unix 惯例的失败码。要"强制执行策略"，必须 `exit 2`。（唯一例外是 `WorktreeCreate`，任何非零退出码都会中止创建。）

但**不是每个事件都能被 exit 2 拦截**。官方给了完整表，几个代表性的：

| 事件 | 能拦截? | exit 2 的效果 |
|---|---|---|
| `PreToolUse` | 是 | 拦截工具调用 |
| `PermissionRequest` | 是 | 拒绝该权限 |
| `UserPromptSubmit` | 是 | 拦截并擦除该提示词 |
| `Stop` / `SubagentStop` | 是 | 阻止 Claude / 子代理停止 |
| `PostToolUse` | 否 | 工具已执行，只能把 stderr 给 Claude 看 |
| `SessionStart` / `Setup` / `Notification` | 否 | stderr 只给用户看，继续执行 |
| `PostToolBatch` | 是 | 在下一次模型调用前停止 agentic loop |
| `PreCompact` | 是 | 阻止压缩 |

### 结构化 JSON 输出（更精细的控制）

退出码只能"拦或不拦"，要更精细的控制就 **exit 0 + 往 stdout 打印 JSON**。官方强调二选一：

> "You must choose one approach per hook, not both: either use exit codes alone for signaling, or exit 0 and print JSON for structured control. Claude Code only processes JSON on exit 0. If you exit 2, any JSON is ignored."
> （每个 hook 必须二选一：要么只用退出码，要么 exit 0 打印 JSON 做结构化控制。Claude Code 只在 exit 0 时处理 JSON；exit 2 时 JSON 被忽略。）

比如 `PreToolUse` 拒绝一次工具调用并说明原因：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

给 `UserPromptSubmit` 注入上下文（注意 `additionalContext` 必须嵌在 `hookSpecificOutput` 里，放顶层会被静默忽略）：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Current branch: release-42. Deploy freeze until Friday."
  }
}
```

`additionalContext` 会被包成一个 system reminder 插到对话里，Claude 下一次模型请求时读到，但不会作为聊天消息显示。它适合放**环境状态、条件性项目规则、外部数据**这类"Claude 应该知道的当前信息"；对永远不变的静态约定，官方建议用 CLAUDE.md（不跑脚本、零成本）。

### 多 hook 同时命中时怎么合并

> "When multiple hooks match the same event, every hook's command runs to completion before Claude Code merges the results. One hook returning `deny` doesn't stop sibling hooks from executing."
> （多个 hook 命中同一事件时，每个 hook 的命令都跑完，然后 Claude Code 才合并结果。一个 hook 返回 `deny` 不会阻止兄弟 hook 执行。）

合并规则：`PreToolUse` 权限决策取**最严格**的答案，顺序是 `deny` > `defer` > `ask` > `allow`；`additionalContext` 则保留每个 hook 的、一起给 Claude。

---

## 六、官方指南里的常见用例

官方指南 `Automate actions with hooks` 给了六个现成场景，每个都带完整配置。挑几个最有代表性的：

### 1. Claude 需要输入时弹桌面通知（Notification）

在 `~/.claude/settings.json` 加 `Notification` 事件，空 matcher 表示所有通知类型都触发：

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

（macOS 用 `osascript -e 'display notification ...'`，Linux 用 `notify-send`；Windows 这段是弹一个消息框。）空 matcher 匹配所有通知类型；也可以精确到 `permission_prompt`（需要你批准工具）、`idle_prompt`（Claude 干完活等你）、`auth_success` 等。

### 2. 每次编辑后自动格式化（PostToolUse）

`PostToolUse` + `Edit|Write` matcher，`jq` 抽出文件路径喂给 prettier。加到项目根 `.claude/settings.json`：

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

### 3. 拦截受保护文件的编辑（PreToolUse + exit 2）

写一个脚本 `.claude/hooks/protect-files.sh`，检查文件路径是否匹配保护模式，匹配则 `exit 2`：

```bash
#!/bin/bash
# protect-files.sh
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

然后在 `.claude/settings.json` 注册（macOS/Linux 需先 `chmod +x`）：

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

### 4. 压缩后重新注入关键上下文（SessionStart + compact matcher）

上下文压缩可能丢掉细节，用 `SessionStart` 的 `compact` matcher 每次压缩后提醒 Claude。脚本写到 stdout 的任何文本都会进 Claude 的上下文：

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

### 5. 自动批准特定权限提示（PermissionRequest + JSON 决策）

有些工具调用你永远会允许，比如 `ExitPlanMode`（计划完成时请求开工）。`PermissionRequest` hook 返回 `"behavior": "allow"` 就等于替你点了允许：

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

官方特别强调：**matcher 尽量收窄**——如果写成 `.*` 或留空，等于自动批准所有权限提示，包括文件写入和 shell 命令。这条 hook 的 transcript 会显示 "Allowed by PermissionRequest hook"。

---

## 七、决策控制速查：每个事件用哪套 JSON 字段

不同事件的 JSON 决策字段**不一样**。官方给了一张速查表，摘录最常用的：

| 事件 | 决策模式 | 关键字段 |
|---|---|---|
| `UserPromptSubmit`、`PostToolUse`、`Stop`、`SubagentStop`、`ConfigChange`、`PreCompact` | 顶层 `decision` | `decision: "block"` + `reason` |
| `PreToolUse` | `hookSpecificOutput` | `permissionDecision`（allow/deny/ask/defer）+ `permissionDecisionReason` |
| `PermissionRequest` | `hookSpecificOutput` | `decision.behavior`（allow/deny） |
| `PermissionDenied` | `hookSpecificOutput` | `retry: true` 告诉模型可以重试被拒的工具调用 |
| `SessionStart`、`Setup`、`SubagentStart` | 只加上下文 | `hookSpecificOutput.additionalContext` |
| `Notification`、`SessionEnd`、`PostCompact` 等 | 无 | 只做副作用（记日志/清理） |

几个值得注意的细节：

- **`PreToolUse` 的四个决策值**：`allow`（跳过交互式确认，但 deny/ask 规则仍然生效）、`deny`（取消调用并把原因给 Claude）、`ask`（正常弹出确认框）、`defer`（仅 `-p` 非交互模式可用，优雅退出进程，让 Agent SDK 包装层收集输入后恢复）。多个 hook 同时返回时优先级 `deny` > `defer` > `ask` > `allow`。
- **hooks 只能收紧、不能放松权限**：hook 返回 `allow` 不覆盖 deny 规则，也不能压制组织设为 `ask` 的连接器工具提示。官方原话：Hooks can tighten restrictions but not loosen them past what permission rules allow.
- **`PreToolUse` 跑在权限模式检查之前**，包括 `dontAsk` 和 `bypassPermissions` 模式——一个返回 `deny` 的 hook 即使在 `--dangerously-skip-permissions` 下也能拦截工具调用。这让你能强制"用户改权限模式也绕不过"的策略。
- **`PostToolUse` 的 `updatedToolOutput`** 可以替换工具返回给 Claude 的输出（脱敏场景），但只改 Claude 看到的，工具**已经实际执行**，副作用无法撤销；替换值必须匹配该工具的输出形状。
- **重写工具输入用 `updatedInput`**：`PreToolUse` 用它改参数再执行，`AskUserQuestion`/`ExitPlanMode` 这类需要用户交互的工具，配合 `allow` + `updatedInput` 可以在非交互模式下"自己回答"。

---

## 八、异步 hook：后台跑，不阻塞 Claude

默认 hook 会阻塞 Claude 直到跑完。长任务（部署、测试套件、外部 API 调用）可以用 `"async": true` 放到后台。官方提醒了代价：

> "Async hooks can't block or control Claude's behavior: response fields like `decision`, `permissionDecision`, and `continue` have no effect, because the action they would have controlled has already completed."
> （异步 hook 无法拦截或控制 Claude 的行为：`decision`、`permissionDecision`、`continue` 这些字段都不生效，因为原本要控制的那个动作早已完成。）

一个"每次 Write 后后台跑测试、跑完把结果回传给 Claude"的例子：脚本把结果写进 `additionalContext`，async hook 完成后的输出会在**下一个对话回合**送达 Claude。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-tests-async.sh",
            "args": [],
            "async": true,
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

约束：只有 `command` 型支持 async；async 的输出要等下一个回合（会话空闲就等下次交互，`asyncRewake` 例外，exit 2 时会立即唤醒 Claude）；async 每次触发都是独立进程，没有去重。

---

## 九、安全：hook 拥有你的完整用户权限

这是官方写得最重的一段，必须原样引用：

> "Command hooks execute shell commands with your full user permissions. They can modify, delete, or access any files your user account can access. Review and test all hook commands before adding them to your configuration."
> （命令型 hook 以你的**完整用户权限**执行 shell 命令。它们可以修改、删除或访问你用户账号能访问的任何文件。在把 hook 命令加入配置前，务必审查并测试。）

官方给出的写 hook 安全准则：

- **校验并净化输入**：永远不要盲目信任输入数据；
- **shell 变量永远加引号**：用 `"$VAR"` 而不是 `$VAR`；
- **拦截路径穿越**：检查文件路径里的 `..`；
- **用绝对路径**：脚本用绝对路径或 `${CLAUDE_PROJECT_DIR}`；
- **跳过敏感文件**：`.env`、`.git/`、密钥等。

另外官方在 `Security` 文档里把 hooks 与权限系统做了分工：

> "CLAUDE.md and permissions solve different problems. CLAUDE.md tells Claude how your project works so it makes good decisions. Permissions and hooks enforce limits regardless of what Claude decides. Use CLAUDE.md for 'we do it this way here.' Use permissions or hooks for security boundaries and anything that must never happen, where you need a guarantee instead of guidance."
> （CLAUDE.md 和权限解决不同的问题：CLAUDE.md 告诉 Claude 你的项目怎么运作，好让它做对决定；权限和 hooks 则**无论 Claude 决定什么都要强制限制**。CLAUDE.md 用于"我们这里是这么做的"，权限/hooks 用于安全边界和"绝对不能发生"的事——那里你需要的是保证，而不是建议。）

这也是 hooks 官方博客式的一句话总结：**"never edit `.env`"写在 CLAUDE.md 里是请求，写成 `PreToolUse` hook 才是执行。**

---

## 十、排错与调试

官方 `Debug your configuration` 和指南的排错章节给了最实用的排查顺序：

1. **hook 不出现** → 跑 `/hooks`：如果定义没列出来，说明没被读取。hooks 必须在 settings 文件的 `"hooks"` key 下，**不在**独立文件里。确认 JSON 合法（不允许尾逗号和注释），确认位置对（项目 hook 在 `.claude/settings.json`，全局在 `~/.claude/settings.json`）。
2. **hook 出现了但不触发** → 大概率是 matcher 的问题：检查 matcher 是否与工具名**精确**匹配（matcher 大小写敏感）；拼错工具名会匹配不到任何东西、静默失败。用 `claude --debug hooks` 启动并触发一次工具调用，debug log 会记录每个事件、哪些 matcher 被检查、hook 的退出码和输出。
3. **转录里出现 "hook error"** → 脚本非零退出。手动测：`echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./my-hook.sh`，然后 `echo $?` 看退出码。看到 "command not found" 就用绝对路径或 `${CLAUDE_PROJECT_DIR}`，或加 `"args": []` 切换到 exec form（不经过 shell 直接 spawn）；看到 "jq: command not found" 就装 jq 或用 Python/Node 解析。
4. **脚本根本没跑** → macOS/Linux 记得 `chmod +x`。
5. **JSON 校验失败** → shell form 的命令 hook 会 spawn `sh -c`（Windows 是 Git Bash），如果 shell profile 里有**无条件 echo**，输出会被塞到 JSON 前面导致解析失败。官方解法是把 profile 里的 echo 包进交互式判断 `if [[ $- == *i* ]]`。

**exec form vs shell form**：`args` 存在时是 exec form——`command` 解析成可执行文件直接 spawn，不经 shell，每个 `args` 元素原样传参（路径含空格/特殊字符无需引号）；省略 `args` 时是 shell form——`command` 字符串交给 shell（sh -c / Git Bash / PowerShell）做分词、变量展开、管道重定向。官方建议：只要 hook 引用了路径占位符（`${CLAUDE_PROJECT_DIR}` 等），优先用 exec form。Windows 上 exec form 要求 `command` 是真 `.exe`，`node_modules/.bin` 里的 `.cmd`/`.bat` shim 不是可执行文件，要用 `"command": "node", "args": ["...eslint.js"]` 这种模式。

超时默认值也要记牢：command/http/mcp_tool 默认 **600 秒**（10 分钟），`UserPromptSubmit` 会把它们降到 30 秒、`MessageDisplay` 降到 10 秒；prompt 默认 30 秒；agent 默认 60 秒；`SessionEnd` 类 hook 共享 **1.5 秒**预算（最多抬到 60 秒）。可以在 handler 上加 `timeout` 字段覆盖。

---

## 十一、hooks vs 其他扩展：什么时候用哪个

官方 `Explore Claude Code features` 把扩展层拆成 CLAUDE.md / Skills / subagents / hooks / MCP / plugins，其中 hooks 的定位是：

> "**Hooks** run your script, HTTP request, prompt, or subagent when Claude Code reaches a lifecycle event."
> （hooks 在 Claude Code 到达某个生命周期事件时运行你的脚本、HTTP 请求、提示词或子代理。）

和 skill 的对比最常被问到：

| 维度 | Hook | Skill |
|---|---|---|
| 触发方式 | 生命周期事件自动触发（`PostToolUse`、`SessionStart`…） | 你敲 `/<name>`，或 Claude 按描述匹配任务后使用 |
| 本质 | **自动化**：每次匹配事件必然执行 | **按需加载的指令/工作流**：进上下文让 Claude 应用 |
| 上下文成本 | 零——除非 hook 返回了输出 | 描述每次会话加载，全文用到了才加载 |
| 适用 | "必须每次一样、不需要 Claude 想"的动作 | "需要 Claude 思考怎么做"的流程 |

官方两句话决策：

> "Use a hook when the action must happen the same way every time and doesn't need Claude to think. For example: format on save, reject `rm -rf /`, post a Slack message when a session ends."
> （当动作必须每次以同样方式发生、且不需要 Claude 思考时用 hook。例如：保存即格式化、拒绝 `rm -rf /`、会话结束时发 Slack 消息。）

> "Put guardrails in hooks. An instruction like 'never edit `.env`' in CLAUDE.md or a skill is a request, not a guarantee. A `PreToolUse` hook that blocks the edit is enforcement."
> （护栏要放在 hooks 里。CLAUDE.md 或 skill 里写"永远别改 `.env`"是请求，不是保证；一个拦截该编辑的 `PreToolUse` hook 才是执行。）

MCP 负责"外连"，插件是"打包层"——一个插件可以同时捆绑 skills、hooks、subagents 和 MCP server。真实组合示例：CLAUDE.md 管项目约定、skill 管部署流程、MCP 连数据库、hook 每次编辑后跑 lint。每种扩展管它最擅长的那件事。

---

## 十二、总结：hooks 的"心智模型"

把官方材料压缩成一张速查表，够你开始写了：

| 问题 | 官方答案 |
|---|---|
| hooks 是什么 | 用户定义的 shell 命令 / HTTP 端点 / LLM 提示词，在生命周期节点自动执行 |
| 为什么用它 | **确定性控制**：某些动作必然发生，不依赖 LLM 想起来 |
| 写在哪儿 | settings.json（全局/项目/本地）、托管策略、插件、skill/agent frontmatter |
| 怎么查看 | `/hooks`（只读浏览器） |
| 输入 | 事件特定 JSON 走 stdin（HTTP hook 走 POST body） |
| 输出 | exit 0（无异议/JSON 决策）、exit 2（拦截）、其他（非阻塞错误） |
| 拦截工具调用 | `PreToolUse` + `exit 2` 或 `permissionDecision: "deny"` |
| 注入上下文 | `additionalContext`（嵌在 `hookSpecificOutput` 里） |
| 自动批准 | `PermissionRequest` + `decision.behavior: "allow"`（matcher 收窄！） |
| 判断型需求 | `type: "prompt"`（单轮 LLM）；`type: "agent"`（多轮验证，实验性） |
| 长任务不阻塞 | `"async": true`（但失去拦截能力） |
| 安全前提 | hook 拥有你完整的用户权限——先审查再配置 |
| 与 CLAUDE.md 的关系 | CLAUDE.md 是请求，hook 是执行；护栏放 hook |

一句话收尾：**凡是你希望"不用想、每次必然发生"的事，就写成一个 hook；凡是你希望"Claude 每次都记住"的事，那不是一个 hook 能保证的——只有 hook 能保证。** 先用 `/hooks` 看现有配置，用官方指南的六个用例做第一个 hook，`exit 2` + `jq` 就是你 90% 场景的全部需要。

---

## 参考来源

本文内容综合以下官方资料整理（均于 2026-08-04 通过 HTTP 请求抓取 `https://code.claude.com` 官方页面获取；先拉取文档索引 `llms.txt` 定位 hooks 相关页面，再抓取各页全文后逐字引用）：

- **Automate actions with hooks（指南）** — https://code.claude.com/docs/en/hooks-guide
  （hooks 定义与"确定性控制"原话、第一个 hook 上手、六个常见用例、matcher 事件表、prompt/agent/HTTP hook、超时与限制、排错技巧、`/hooks` 菜单）
- **Hooks reference（参考）** — https://code.claude.com/docs/en/hooks
  （事件生命周期与"三种节奏"、31 个事件的触发时机与 exit 2 行为表、三级配置结构、五种 hook 类型字段、matcher 求值规则、stdin 输入/退出码/JSON 输出协议、决策控制速查表、`if` 字段 Bash 匹配表、exec form vs shell form、路径占位符、异步 hook、安全注意事项、Windows PowerShell、debug 日志）
- **Security** — https://code.claude.com/docs/en/security
  （hook 拥有完整用户权限的免责声明、权限系统与 hooks 的分工、用 ConfigChange hook 审计配置变更的团队安全实践）
- **Explore Claude Code features** — https://code.claude.com/docs/en/features-overview
  （hooks vs skill 对比表、"Use a hook when…"与"Put guardrails in hooks"原话、各扩展的上下文成本）
- **Debug your configuration** — https://code.claude.com/docs/en/debug-your-config
  （`/hooks` 排查流程、matcher 拼写错误导致静默失败、`claude --debug hooks` 观测法）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（CLAUDE.md 与上下文管理）、`Agent/ClaudeCode/Claude Code 权限模式（permission modes）官方说明.md`（权限系统与 hooks 的配合）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
