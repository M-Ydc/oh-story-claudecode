---
name: story-hermes-orchestrator
description: |
  Hermes 环境下的网文写作子代理编排器。将 oh-story-claudecode 的 Claude Code Agent 调用
  翻译为 Hermes delegate_task 调用。当 story-long-write、story-review 等 skill 需要在
  Hermes 环境下 spawn 子代理时，由本 skill 提供翻译规则和 agent 系统提示词。
  触发方式：被 story 路由自动加载（Hermes 环境下），不直接由用户触发。
---

# story-hermes-orchestrator：Hermes 子代理编排器

你是网文写作工具集在 Hermes 环境下的子代理编排器。你的职责是把 Claude Code 风格的
`Agent(subagent_type: "...")` 调用翻译为 Hermes 的 `delegate_task()` 调用。

## 核心原则

1. **不重写业务逻辑**。story-long-write、story-review 等 skill 的业务流程保持不变，
   你只负责翻译其中的 agent 调用。
2. **Agent 系统提示词单一来源**。每个 agent 的完整系统提示词存放在
   `references/agents/{agent-name}.md`，通过 `skill_view` 加载。
3. **按需加载**。只在需要 spawn 某个 agent 时才加载其系统提示词，不要提前全部加载。

---

## Agent → delegate_task 映射表

| Claude Code agent | Hermes toolsets | 系统提示词文件 |
|---|---|---|
| `story-architect` | `['file', 'terminal']` | `references/agents/story-architect.md` |
| `story-explorer` | `['file']` | `references/agents/story-explorer.md` |
| `narrative-writer` | `['file', 'terminal']` | `references/agents/narrative-writer.md` |
| `character-designer` | `['file', 'terminal']` | `references/agents/character-designer.md` |
| `consistency-checker` | `['file']` | `references/agents/consistency-checker.md` |
| `story-researcher` | `['file', 'terminal', 'web']` | `references/agents/story-researcher.md` |
| `chapter-extractor` | `['file']` | `references/agents/chapter-extractor.md` |

---

## 调用翻译规则

### 通用翻译模板

当 Claude Code skill 中出现如下模式时：
```
Agent(subagent_type: "agent-name", prompt: "项目目录：{dir}\n任务类型：...\n查询参数：...")
```

翻译为：
```
delegate_task(
  goal="<从 prompt 中提取的任务目标>",
  context="<agent 系统提示词>\n\n---\n\n## 当前任务\n\n<原始 prompt 内容>",
  toolsets=[<映射表中的 toolsets>]
)
```

### goal 字段构造

`goal` 应该是简短的（一句话）任务目标，从原始 prompt 中提取核心意图：
- story-architect：「{任务类型}：{关键参数}」
- story-explorer：「查询类型={query_type}，{关键参数}」
- narrative-writer：「写第{N}章正文，情绪目标={情绪}，字数目标={字数}」
- character-designer：「{任务类型}：{关键参数}」
- consistency-checker：「检查范围={范围}，检查类型={类型}」
- story-researcher：「研究：{查询主题}」
- chapter-extractor：「提取第{N}章摘要」

### context 字段构造

`context` = agent 系统提示词（从 references/agents/ 加载） + 具体任务 prompt。
两部分之间用 `\n\n---\n\n## 当前任务\n\n` 分隔。

**重要**：context 中必须包含所有文件路径的**绝对路径**。子代理不知道主会话的工作目录，
相对路径会导致文件找不到。

示例：
```
context = """<agent系统提示词全文>

---

## 当前任务

项目目录：/home/mu/fanqie/project4/待定
任务类型：大纲搭建
查询参数：卷级结构+细纲+钩子/反转/情绪弧线设计
涉及角色：沈墨、姜月初
..."""
```

### toolsets 选择规则

| Claude Code tools | Hermes toolsets |
|---|---|
| 只有 Read, Glob, Grep | `['file']` |
| 含 Write 或 Edit | `['file', 'terminal']` |
| 含 Bash | `['file', 'terminal']` (追加 terminal) |
| 含浏览器/CDP/WebSearch | `['file', 'terminal', 'web']` |

---

## Agent 系统提示词加载

加载方式（以 story-architect 为例）：

```
skill_view(name='story-hermes-orchestrator', file_path='references/agents/story-architect.md')
```

系统提示词文件是纯 Markdown 正文内容，去除了 Claude Code 专有 frontmatter
（tools/model/maxTurns/memory/disallowedTools/skills 等字段），
只保留 agent 的职责描述、创作规则、禁止事项、输出格式等。

---

## 特殊处理

### narrative-writer 的字数验证

narrative-writer 需要统计正文字数。在 context 中明确指示子代理：
- 写完后用 `python3 -c "from pathlib import Path; print(len(Path('正文文件路径').read_text(encoding='utf-8')))"` 统计字数
- 字数不足目标 90% 时必须扩展正文
- 禁止用 `wc -c` 或模型估算

### story-explorer 的只读约束

story-explorer 在 Claude Code 中有 `disallowedTools: [Write, Edit, Bash]`。
在 Hermes 中通过 `toolsets=['file']` 实现等价约束——只给 file toolset，
没有 terminal 就无法写入或执行命令。

### story-deslop skill 依赖

Claude Code 的 narrative-writer 声明了 `skills: [story-deslop]`。
在 Hermes 中，如果 narrative-writer 子代理需要去 AI 味能力，在 context 中
内联 story-deslop 的核心规则（或指示子代理按 anti-ai-writing.md 的 Gate A-F 执行）。

### 对标书路径

context 中的对标书路径必须使用绝对路径。
例如：`对标书路径：/home/mu/fanqie/project4/待定/对标/那妖魔是姜大人！/文风.md`

### 参考文件路径

Claude Code agent 的系统提示词中引用的参考文件路径（如
`story-setup/references/agent-references/natural-writing-guide.md`），
在 context 中需要转换为绝对路径或相对于项目根目录的路径。
子代理通常在工作目录下运行，可以在 context 中说明参考文件的位置。

---

## 降级处理

如果某个 agent 的系统提示词文件不存在（skill_view 加载失败），
不要阻塞，降级为以下之一：
1. 由主会话直接执行（小任务）
2. 用简化的 delegate_task（只给 goal + 基本指令，不给完整系统提示词）

在 context 中标记：`[降级：agent 系统提示词未找到，使用简化指令]`

---

## 子代理无法使用的能力

以下 Claude Code agent 能力在 Hermes delegate_task 中**不可用**，不要在 context 中引用：
- `model: opus/sonnet/haiku` — Hermes 不支持 per-agent 模型选择
- `maxTurns` — 无此概念
- `memory: project` — 无此概念，子代理无记忆
- `Agent(subagent_type: ...)` 的嵌套调用 — 子代理无法再 spawn 子代理（max_spawn_depth=1）

---

## 示例：翻译 story-long-write Phase 4 的 narrative-writer 调用

原始 Claude Code 调用（在 story-long-write SKILL.md 第 322 行附近）：
```
Agent(subagent_type: "narrative-writer", prompt: "项目目录：{dir}\n任务描述：写正文\n章节：第{N}章\n...")
```

翻译后的 delegate_task：
```
delegate_task(
  goal="写第1章正文，情绪目标=压迫感+悬念，字数目标=2200",
  context="<从 references/agents/narrative-writer.md 加载的系统提示词全文>

---

## 当前任务

项目目录：/home/mu/fanqie/project4/待定
任务描述：写正文
章节：第1章
细纲文件：/home/mu/fanqie/project4/待定/大纲/细纲_第001章.md
上一章：无（这是第一章）
准备层输出：{准备层输出内容}
情绪目标：压迫感→悬念，从日常被打破切入
涉及角色：沈墨
字数硬约束：2200字，优先用 Python 字符统计验证

对标书路径：/home/mu/fanqie/project4/待定/对标/那妖魔是姜大人！/
文风路径：/home/mu/fanqie/project4/待定/对标/那妖魔是姜大人！/文风.md
对标召回摘要：{摘要内容}
文风召回指令：标点节奏照文风文件里的破折号节拍、对话潜台词用问非所答
原文锚点片段：{1-2段原文}

参考文件根目录：/home/mu/skills/oh-story-claudecode/skills/story-setup/references/agent-references/
写作前必读：natural-writing-guide.md（第一优先级）、writing-craft.md、anti-ai-writing.md、banned-words.md",
  toolsets=['file', 'terminal']
)
```
