# Claude Code 是否建议使用 codegraph 等 RAG 工具？

> **一句话总结**：**官方并不建议为了"让 Claude 更懂代码"而给 Claude Code 额外加一层 embedding / 向量索引式的 RAG 工具。** Claude Code 本身就是一套"agentic search（智能体式检索）"：实时遍历文件系统、grep、按需读文件，官方明确说**没有 embedding 流水线、没有集中式索引**。对大代码库，官方推荐的解法是分层 CLAUDE.md + LSP 代码智能插件 + subagent，而不是预先建一个代码索引；只有当你的组织**已经**有一套 code search / RAG 索引时，官方才认可把它**以 MCP server 形式**暴露给 Claude 查询。
>
> 本文基于 Claude Code 官方文档 `How Claude Code works`、`Best practices for Claude Code`、`Set up Claude Code in a monorepo or large codebase`、`Discover and install prebuilt plugins`、`Explore the context window`，以及官方 Claude 博客 `How Claude Code works in large codebases` 整理，文末附参考来源。

很多人在把 Claude Code 接入大项目时会遇到一个"标配问题"：要不要像用 Cursor / Continue / Aider 那样，先跑一个 codegraph、sourcegraph、repo-map 之类的工具，把整个代码库嵌入成向量索引，再让 Claude 去"检索上下文"？

直觉上很合理——代码库太大，Claude 会不会"找不全"？预先索引一下不就能让它"看得更多"？

但官方文档和博客给出的答案，和这种直觉恰好相反。

---

## 一、先搞清楚：Claude Code 找代码，靠的不是 RAG

RAG（检索增强生成）类工具的工作方式是：**先把整个代码库分块、embedding 成向量索引，查询时检索最相关的若干 chunk**。

而 Claude Code 的架构**不是**这样。官方博客明确对比了这两种路线：

> "RAG-powered AI coding tools work by embedding the entire codebase and retrieving relevant chunks at query time."
> （RAG 驱动的 AI 编程工具，工作方式是把整个代码库 embedding，查询时检索相关的 chunk。）

而 Claude Code 走的是另一条路——官方博客的原文：

> "Claude Code navigates a codebase the way a software engineer would: it traverses the file system, reads files, uses grep to find exactly what it needs, and follows references across the codebase."
> （Claude Code 像一个软件工程师一样在代码库里导航：遍历文件系统、读文件、用 grep 找到恰好需要的内容、跨代码库追踪引用。）

官方文档 `How Claude Code works` 把内置工具分成五类，其中"搜索"与"代码智能"这两类的定位是：

| 工具类别 | 做什么 |
|---|---|
| **文件操作** | 读文件、编辑、新建、重命名 |
| **搜索** | "按模式找文件、用正则搜内容、探索代码库"（即内置 Grep / Glob） |
| **执行** | 跑命令、起服务、跑测试、用 git |
| **Web** | 搜索网页、抓文档 |
| **代码智能** | "编辑后看到类型错误与告警、跳到定义、查找引用"（需要代码智能插件） |

注意：这份内置工具清单里**没有**"查向量库""检索 embedding"这一类。Claude Code 的默认方式是**它自己决定读哪些文件**——基于你的提示词、它之前读到的内容，在实时文件系统上逐个去读。

顺着全站文档索引（`llms.txt`，当前约 174 个页面）做关键词扫描，同样找不到任何 RAG / embedding / 向量索引的专页——LSP 只作为"代码智能"出现在 `large-codebases` 与插件文档里。**"官方文档没给 RAG 开专页"本身就是个信号**：它不是 Claude Code 的核心方案。

## 二、为什么官方"故意"不用 embedding 索引？——索引会过期

这是最关键的一点：官方不是没想到 RAG，而是**明确论证了在大型活跃代码库上，embedding 索引是一种会失效的方案**。官方博客原文：

> "At large scale, those systems can fail because embedding pipelines can't keep up with active engineering teams."
> （在大规模下，这类系统会失败，因为 embedding 流水线跟不上活跃工程团队的节奏。）

> "By the time a developer queries the index, it reflects the codebase as it previously existed weeks, days, or even hours before."
> （当开发者查询索引时，索引反映的是代码库**过去**的样子——几周前、几天前，甚至几小时前。）

> "Retrieval then returns a function the team renamed two weeks ago, or references a module that was deleted in the last sprint, with no indication that either is out of date."
> （检索出的可能是团队两周前重命名过的函数，或上个迭代已经删掉的模块，而检索结果**看不出这两者都已过时**。）

索引从"构建"到"被查询"之间天然存在滞后，而代码是每分钟都在变的。Claude Code 的应对方式是反着来：

> "Agentic search avoids those failure modes. There's no embedding pipeline or centralized index to maintain as thousands of engineers commit new code. Each developer's instance works from the live codebase."
> （智能体式检索避免了这些失败模式：**没有需要维护的 embedding 流水线，也没有集中式索引**——哪怕成千上万的工程师在不断提交新代码。每个开发者的实例都工作在**实时**代码库上。）

官方还特别强调了"不需要索引"这一点的运维价值：

> "It operates locally on the developer's machine and doesn't require a codebase index to be built, maintained, or uploaded to a server."
> （它在开发者本机上本地运行，不需要构建、维护、或上传一个代码库索引到服务器。）

换句话说：**你越是想用"建索引"来解决"Claude 找不到代码"，索引过期这个新问题就越会反噬你。** Claude Code 的答案不是"更好的索引"，而是"不索引、实时查"。

## 三、那"大代码库找不到东西"怎么解？官方给了四条路

官方并非两手一摊。在 `Set up Claude Code in a monorepo or large codebase` 文档里，对大仓库的解法是**让 Claude 只看到它该看的部分**，而不是让它"看到更多"：

### 1. 分层 CLAUDE.md，而不是一份巨大的索引

官方博客把大仓库的上下文组织概括为"用 CLAUDE.md 文件与 skills 分层加载"（layering context with CLAUDE.md files and skills），并强调：

> "Claude loads them additively as it moves through the codebase."（Claude 边在代码库中移动，边**按需累加**加载这些文件。）

分工上，博客建议"根文件管全局、子目录文件管局部约定"（root file for the big picture, subdirectory files for local conventions）——并不是给 Claude 一份"整个代码库的索引"，而是让它在进入哪个目录时只加载那一部分的约定。

原理：**上下文窗口是稀缺资源**。让 Claude 一次性"看到"整个代码库的索引，等于把最贵的资源浪费在它当前任务用不到的地方。分层的 CLAUDE.md 恰恰相反——只把当前目录相关的约定加载进来。

### 2. LSP 代码智能插件——官方推荐的"更精准的查找"

这是官方文档反复推荐、且最接近"codegraph 想要解决的事"的替代方案：

> "In a large codebase, finding where a symbol is defined or used can cost many file reads and grep calls. Code intelligence plugins connect Claude to a language server so it can jump to definitions, find references, and surface type errors directly instead of scanning the tree."（在大型代码库里，找一个符号的定义或使用点，可能要花掉大量文件读取和 grep 调用。代码智能插件把 Claude 接到语言服务器上，让它**直接**跳转定义、找引用、暴露类型错误，而不是整棵树的扫描。）

官方博客进一步说明了它对比"文本检索"的价值：在大型代码库里 grep 一个常见函数名会返回海量匹配，而 LSP 只按**符号**检索——

> "Surfacing this to Claude gives it symbol-level precision: it can follow a function call to its definition, trace references across files, and distinguish between identically named functions in different languages."（把这套能力暴露给 Claude，它就有了**符号级精度**：能顺着一个函数调用追到它的定义、跨文件追踪引用、区分不同语言里同名函数。）

> "LSP returns only the references that point to the same symbol, so the filtering happens before Claude reads anything."（LSP 只返回指向**同一个符号**的引用，过滤发生在 Claude 读任何东西之前。）

插件文档 `Discover and install prebuilt plugins` 对 LSP 能力给出了同样的定位，直接点破它对比 grep 的优势：

> "These operations give Claude more precise navigation than grep-based search."（这些操作给 Claude 带来比基于 grep 的搜索更精准的导航。）

对照关系很清晰：

| 手段 | 检索粒度 | 是否官方推荐 |
|---|---|---|
| embedding / RAG 索引（codegraph 等） | 文本 chunk，靠相似度 | 不推荐，索引会过期 |
| grep / glob（内置） | 文本正则 | 默认机制，够用即可 |
| **LSP 代码智能插件**（`/plugin install typescript-lsp@claude-plugins-official` 等） | **符号级**：定义/引用/类型 | **推荐**，专为大仓库设计 |

安装方式是 `/plugin install <语言>-lsp@claude-plugins-official`，官方市场提供了 C/C++、C#、Go、Java、Kotlin、Lua、PHP、Python、Rust、Swift、TypeScript 等语言的插件。

### 3. subagent 隔离探索，别污染主上下文

`Explore the context window` 文档里有一个很直观的数字对比：一个研究型 subagent 读了 6,100 token 的文件，回传给你的主会话只有 420 token 的结果——"**That's the context savings.**"（这就是上下文节省量。）

官方博客也提到："some teams spin up a read-only subagent to map a subsystem and write findings to a file."（有些团队会起一个只读 subagent 去绘制子系统的地图，并把发现写进文件。）

### 4. MCP 接入——官方文档里唯一把 RAG 写成解法的位置

`large-codebases` 文档里，有一句是全文**唯一**出现"RAG"字样的官方表态，非常值得读细：

> "MCP servers: if your organization already runs a code search or RAG index over the repository, expose it as an MCP tool so Claude queries it instead of reading files directly."（MCP server：如果你的组织**已经**在跑一个代码搜索或 RAG 索引，就把它暴露成一个 MCP 工具，让 Claude 去**查询它**，而不是直接读文件。）

注意这个句子的两个前提：**"already"（已经有）**、以及用途是"查询"而不是"塞进上下文"。官方并没有把 RAG 当作"应该去搭建的东西"，而是说：如果你恰好已经投资了这么一套基础设施，别浪费，用 MCP 让 Claude 能查到它。

## 四、结论：codegraph 这类 RAG 工具，到底用不用？

把官方态度翻译成可执行的建议，大致是下面这张表：

| 场景 | 官方倾向 | 理由 |
|---|---|---|
| 中小型 / 整洁的代码库 | **不用装** | Claude 自带 grep/glob + 按需读文件已足够；多加一层 embedding 是冗余 |
| 大型代码库、Claude 频繁"找不到符号" | **先装 LSP 代码智能插件** | 官方明确推荐的替代品：符号级导航比任何文本检索都精准 |
| 巨型 monorepo，探索开销失控 | **分层 CLAUDE.md + subagent + 限定启动目录** | 上下文是稀缺资源，解法是"少看"，不是"看得更多" |
| 你/组织**已经有**现成代码搜索索引（Sourcegraph 等） | **暴露成 MCP server** | 官方唯一认可 RAG 的姿势：查询，而不是索引替换 |
| 想靠"建索引"解决一切 | **别** | 索引落后于活跃代码库是官方点名的失败模式 |

几个容易踩的坑，值得单独强调：

- **别把"embedding 索引"和"符号/图索引"混为一谈。** codegraph、Sourcegraph 这类工具本质更接近"符号/依赖图"，而 Cursor 的 @Codebase、Continue、Aider 的 repo-map 这类是"chunk embedding"。前者和官方推荐的 LSP 插件**功能重叠**，后者正是官方博客批评的那条路线。如果你已经装了 LSP 代码智能插件，codegraph 能补的东西其实很有限。
- **RAG 工具喂给 Claude 的是"检索结果"，不是"文件本身"。** 官方 `large-codebases` 文档那句 MCP 建议也点出了这点：让 Claude 查询索引，替代的是它"读文件"这一动作。但检索结果往往丢掉了文件的结构、上下文和跨文件关系——而这恰恰是 agentic search 引以为傲的东西。
- **别用索引绕过上下文管理。** 上下文窗口是 Claude Code 最需要管理的资源（官方原话："The context window is the most important resource to manage"）。一个把"整个代码库摘要"塞进上下文的工具，等于反向操作。

## 五、务实的三步走

如果现在就要给一个"开工建议"，我建议按顺序做这三件事，每做一步觉得够用就停：

1. **先用默认能力。** 直接在提示词里指文件（`@` 引用）、指定目录、让 Claude 自己 grep/读文件。多数代码库到这步就够了。
2. **遇到"找不到"再装 LSP 插件。** `claude` 里跑 `/plugin install <语言>-lsp@claude-plugins-official`，配合分层 CLAUDE.md 和 subagent 探索。
3. **真到了"组织级巨型仓库"才谈 RAG。** 只有当你已经在维护一套代码搜索/索引基础设施时，才考虑用 MCP 把它暴露给 Claude——并且把它定位成"查询补充"，而不是 Claude 找代码的主要方式。

---

## 参考来源

本文内容综合以下资料整理（均于 2026-08-04 通过 web_fetch 获取）：

- **How Claude Code works** — https://code.claude.com/docs/en/how-claude-code-works
  （agentic loop 三阶段、内置工具五分类：文件操作/搜索/执行/Web/代码智能、上下文窗口管理）
- **Set up Claude Code in a monorepo or large codebase** — https://code.claude.com/docs/en/large-codebases
  （分层 CLAUDE.md、claudeMdExcludes、Read deny 规则、LSP 代码智能插件、sparse worktree、以及唯一提到 RAG 的 MCP 接入建议）
- **Discover and install prebuilt plugins** — https://code.claude.com/docs/en/discover-plugins
  （代码智能插件清单：各语言 LSP 插件与所需二进制、跳转定义/查找引用/类型诊断能力）
- **Explore the context window** — https://code.claude.com/docs/en/context-window
  （会话启动加载项、每次文件读取的 token 成本、subagent 的上下文节省量——"subagent 读了 6,100 token、主会话只收到 420 token 结果"）
- **Best practices for Claude Code** — https://code.claude.com/docs/en/best-practices
  （"The context window is the most important resource to manage" 原话；用 subagent 做 investigation、给 Claude 验证手段等官方最佳实践）
- **Manage costs effectively** — https://code.claude.com/docs/en/costs
  （上下文管理、减少 token 用量的官方策略）
- **Claude 官方博客：How Claude Code works in large codebases** — https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start
  （发布于 2026-05-14；RAG vs agentic search 的直接对比、embedding 流水线失效的论证、"没有 embedding 流水线/集中式索引"、LSP 与 subagent 实践）

> 相关文档：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`（上下文窗口与 CLAUDE.md 管理）、`Agent/ClaudeCode/让web_fetch生效.md`（web 抓取与代理配置）。
