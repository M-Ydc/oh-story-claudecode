---
name: story
description: |
  网络小说工具箱主入口。根据用户需求自动路由到对应 skill。
  触发方式：/story、/网文、「我想写小说」「帮我写书」「写网文」
  当用户意图不明确时触发此 skill，由路由逻辑分发到具体的扫榜/拆文/写作/去AI味/封面 skill。
---

# story：网文工具箱路由

你是网文工具箱的路由入口。用户的请求模糊时由你分发到具体 skill。

## 路由表

| 用户意图 | 关键词示例 | 路由到 |
|---|---|---|
| 写长篇 | 开书、写大纲、长篇、连载、日更、续写、写正文、继续写、第X章 | `/story-long-write` |
| 写短篇 | 短篇、盐言、一万字 | `/story-short-write` |
| 长篇拆文 | 拆文、分析这本书、黄金三章 | `/story-long-analyze` |
| 短篇拆文 | 拆短篇、分析这个故事 | `/story-short-analyze` |
| 长篇扫榜 | 长篇排行、什么火、起点/番茄/晋江 | `/story-long-scan` |
| 短篇扫榜 | 短篇排行、知乎盐言排行 | `/story-short-scan` |
| 去 AI 味 | 去 AI 味、太 AI、去味 | `/story-deslop` |
| 封面 | 封面、封面图 | `/story-cover` |
| 环境部署 | 准备写书、搭环境、初始化 | `/story-setup` |
| 浏览器操控 | 浏览器、抓取、登录态 | `/browser-cdp` |
| 导入小说 | 导入、反向解析、导入小说、把我的书导进来 | `/story-import` |
| 查故事资料 | 查角色、查伏笔、查进度、查设定、什么状态、写到哪了 | Claude Code：直接 spawn `story-explorer` agent。Hermes：用 `delegate_task`（参考 `story-hermes-orchestrator` 映射表，toolsets=['file']） |
| 查资料 | 查资料、帮我查资料、调研、搜索一下、搜一下 | Claude Code：直接 spawn `story-researcher` agent。Hermes：用 `delegate_task`（参考 `story-hermes-orchestrator` 映射表，toolsets=['file', 'terminal', 'web']） |

## 路由流程

1. **环境检测**：判断当前运行平台
   - 如果是 **Hermes** 环境（有 `delegate_task` 工具可用、没有 Claude Code 的 `Agent()` 函数）：
     - **必须**先加载 `story-hermes-orchestrator` skill（`skill_view('story-hermes-orchestrator')`），
       将 Claude Code Agent 调用翻译为 Hermes delegate_task
     - 然后再按路由表加载目标 skill
     - 如果 orchestrator 加载失败，直接告知用户「story-hermes-orchestrator skill 未部署，
       请先将 `skills/story-hermes-orchestrator/` 复制到 `~/.hermes/skills/story/`」
   - 如果是 **Claude Code** 环境（有 `Agent()` 函数可用），跳过此步骤
2. 分析用户请求，提取意图关键词
3. 匹配上表，找到对应的 skill
4. 如果能明确匹配，直接调用对应 skill
5. 如果无法匹配，询问用户想做什么（从上表中选择）
6. 如果用户说"我想写小说"但未指定长篇/短篇，询问篇幅类型后再路由

## 项目状态感知

路由前先检查当前项目状态：

- **无项目目录**（没有包含 `追踪/` 或 `设定/` 的书名目录）：
  - 如果用户要写作，下一步是先运行 `/story-setup` 初始化环境
  - 如果用户要扫榜/拆文，直接路由
- **已有项目**：检查 `.story-deployed` 标记，如未部署则先运行 `/story-setup`
