---
name: claude-code-best-practices
description: 调研 Claude Code 官方文档并撰写知识库博客/笔记。当用户要求查"Claude Code 官方文档对某话题/某工具怎么说的"、写 Claude Code 相关 blog 或笔记、回答"Claude Code 是否推荐某工具/做法"、或需要以官方文档为准整理 Claude Code 知识时，务必使用本技能——即使没明说"写 blog"，只要涉及 Claude Code 官方立场调研或知识库写作，都应调用本技能获取调研流程（web_fetch + llms.txt 索引 + 原话引用核对）与知识库 house style。
---

# Claude Code 官方文档调研与知识库博客写作最佳实践

> **一句话总结**：写 Claude Code 相关 blog 的正确姿势是**以 web_fetch 抓官方文档为主、先抓文档索引、引用只引原话、结论如实标注依据**，而不是依赖 WebSearch、猜文档 URL、或凭记忆复述官方立场。
>
> 本文沉淀自一次完整实践——调研"Claude Code 是否推荐 codegraph 等 RAG 工具"并成文（`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`）。所列坑与解法均为当次实测。

很多人拿到"写一篇关于 Claude Code 的 blog / 回答官方对某工具怎么看"这类任务，会下意识走"搜索 → 抓几页 → 凭印象写"的流程。这套流程在 Claude Code 官方文档这个特定对象上，踩坑概率极高。下面按"流程 → 避坑 → 环境 → 素材 → 风格"五块讲透。

> 何时**不需要**本技能：纯闲聊、非 Claude Code 话题、或用户只要一句简短答案（此时快速 web_fetch 一两页即可，不用走完整流程与 house style）。本技能服务于"以官方文档为准、产出成文产物"的场景。

---

## 一、流程总览（按此顺序执行）

1. **先抓官方文档索引**：`https://code.claude.com/docs/llms.txt`，一次性拿到全站约 180 个页面的标题 + URL。
2. **用索引做关键词扫描**：让 web_fetch 在索引里扫你关心的话题词（如 `RAG`、`embedding`、`code intelligence`、`index`），定位所有相关页——或得出"官方文档根本没提"这个结论。
3. **并行抓取相关页**：对彼此独立的页面用并行 WebFetch，减少往返。
4. **跟文档页外链的官方博客**：核心表态常在 `claude.com/blog/...`，文档页末尾会链出。
5. **提取逐字引用并交叉核对**，区分"原话"与"转述"。
6. **按 house style 成文**，文末列参考来源（含 URL、获取方式与日期）。

## 二、避坑清单（本次实测）

| 坑 | 表现 | 解法 |
|---|---|---|
| **依赖 WebSearch** | 部分环境 WebSearch 直接报错（本次即报 `Thinking mode does not support this tool_choice`） | 以 web_fetch 为主；"找资料"先抓官方索引而非搜索 |
| **猜文档 URL** | 猜 `/docs/en/context` → 404，浪费一次往返 | 一律先抓 `llms.txt` 索引再定 URL；主页会提示抓索引 |
| **一篇篇试页面** | 想找"官方对 RAG 怎么说"，试了多页无果 | 让 web_fetch 扫 `llms.txt` 关键词，一次定位所有相关页 |
| **忽略"官方没提"** | 扫完发现官方文档根本没有 RAG/embedding 专页 | "搜遍文档都没提"本身就是高价值结论，要写进 blog |
| **只看文档页** | 文档页对某话题只给一句（如 RAG 仅在 `large-codebases` 出现一次） | 文档页末尾外链的官方博客才是核心表态，必须跟链接 |
| **大输出被截断** | web_fetch 结果过大（> 数十 KB）被持久化到 `tool-results/call_*.txt`，只返回前 2KB 预览 | 用 Read / grep 该文件提取所需段落，不要重抓整页 |
| **引用失真** | web_fetch 的小模型总结可能截断 / 转述原话 | 逐字引用前核对原文；拿不准就写成转述，宁缺毋滥 |
| **夸大结论** | 把"博客/文档推断"说成"官方明说" | 如实标注依据（哪篇文档/博客、哪句原话），文末列出全部来源 |

## 三、环境前提：web_fetch 必须能通

本机 web_fetch 依赖代理配置，详见 `Agent/ClaudeCode/让web_fetch生效.md`：

- `settings.json` 的 `env` 块写入 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`（端口按实际 clash 端口，经典 7890），并配 `NO_PROXY`。
- `settings.json` 根节点加 `"skipWebFetchPreflight": true`。

不配好这两项，web_fetch 可能直接失败或走 preflight 拦截，一切流程无从谈起。

## 四、常用官方 URL（素材库）

- 文档索引：https://code.claude.com/docs/llms.txt
- Best practices for Claude Code：https://code.claude.com/docs/en/best-practices
- How Claude Code works：https://code.claude.com/docs/en/how-claude-code-works
- 大仓库 / monorepo：https://code.claude.com/docs/en/large-codebases
- 插件（含代码智能 LSP）：https://code.claude.com/docs/en/discover-plugins
- 上下文窗口：https://code.claude.com/docs/en/context-window
- 成本与 token：https://code.claude.com/docs/en/costs
- Memory：https://code.claude.com/docs/en/memory
- Skills：https://code.claude.com/docs/en/skills
- 官方博客：https://claude.com/blog/（文档页外链，如 `how-claude-code-works-in-large-codebases-best-practices-and-where-to-start`）

## 五、知识库 house style

`Agent/ClaudeCode/` 下的 blog 统一结构（范例见同目录两篇既有文档）：

- 开头 `> **一句话总结**` blockquote，一句话讲清结论；并注明"本文综合哪些官方资料、文末附参考来源"。
- 中文分节标题（一、二、三…）；官方原话用 blockquote 引用，给出**英文原文 + 中文解释**。
- 用**对比表格**讲清工具/策略/前后差异（三列及以上）。
- 文末 `## 参考来源`：逐条列出 URL、**获取方式与日期**（如"均于 2026-08-04 通过 web_fetch 获取"）、每篇覆盖了什么；结尾附相关文档链接（用 `Agent/ClaudeCode/...` 相对路径）。
- 文件名用空格分词的中文标题，如 `Claude Code 是否建议使用 codegraph 等 RAG 工具.md`。

## 六、交付检查清单（成文前自查）

- [ ] 结论有据可查：每处"官方原话"都能指到来源，未凭记忆编造
- [ ] 用 web_fetch 抓过官方文档（含 `llms.txt` 索引定位），非 WebSearch/猜测 URL
- [ ] 文末"参考来源"列全：URL、获取方式与日期、每篇覆盖什么
- [ ] 符合 house style：一句话总结开头、中英对照引用、对比表格、中文分节
- [ ] 若官方文档/博客"没提"某话题，已如实写明，而非强行断言

## 七、参考来源

本 SKILL 沉淀自以下实践（2026-08-04）：

- 成文产物：`Agent/ClaudeCode/Claude Code 是否建议使用 codegraph 等 RAG 工具.md`
- 风格参考：`Agent/ClaudeCode/Claude Code 提示词最佳实践.md`
- 环境参考：`Agent/ClaudeCode/让web_fetch生效.md`
- 官方格式定义：https://code.claude.com/docs/en/skills （SKILL.md 结构：frontmatter `name`/`description` + markdown 正文）
