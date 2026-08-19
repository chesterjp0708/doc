# Agent Harness 设计实践（无 API 环境）

## 背景

很多企业环境下：

- 无法申请 OpenAI API
- 无法直接使用 Azure OpenAI
- 只能使用：
  - VS Code Copilot
  - Claude Code
  - Cursor
  - Gemini Code Assist

这类 IDE 内置 AI。

此时无法构建真正意义上的 Autonomous Agent（自治 Agent），因为 AI 无法自动循环执行：

```text
LLM
 ↓
Tool Call
 ↓
Observe
 ↓
Reason
 ↓
Repeat
```

但仍然可以构建一个：

```text
Human-in-the-Loop Agent Harness
```

实现：

- 长期记忆
- 任务规划
- 决策管理
- 项目状态管理
- AI 无关（AI-Agnostic）工作流

---

# 核心原则

## 不要管理 AI

错误思路：

```text
Claude Harness
Copilot Harness
Cursor Harness
```

结果：

```text
Claude记得A
Copilot记得B
Cursor记得C
```

形成多个记忆源。

## 要管理项目

正确思路：

```text
Project
  ↑
Harness

Claude
Copilot
Cursor
Gemini
```

即：

```text
AI只是客户端
Harness属于项目
```

---

# Single Source of Truth

所有 Agent 共享同一套上下文。

```text
project/
│
├── src/
├── docs/
│
├── .ai/
│   ├── START.md
│   ├── rules.md
│   ├── state.yaml
│   ├── memory.md
│   ├── todo.md
│   ├── decisions.md
│   └── journal/
│
└── adr/
```

---

# 文件职责设计

## START.md

Agent 启动入口。

```markdown
# Project Bootstrap

进入项目后必须执行：

1. 阅读 rules.md
2. 阅读 state.yaml
3. 阅读 architecture.md
4. 阅读 todo.md
5. 总结当前状态
6. 给出下一步建议

未经完成上述步骤禁止修改代码。
```

作用：统一初始化流程。

## rules.md

Agent 工作规范。

```markdown
# AI Rules

允许更新：
- memory.md
- todo.md
- journal

禁止直接修改：
- architecture.md

重大设计变更：
必须新增 ADR

完成任务后：
更新 Journal
更新 State
更新 Todo
```

作用：统一所有 AI 行为。

## state.yaml

项目实时状态。

```yaml
project:
  name: AgentHarness

phase: development

current_task:
  id: TASK-023

progress: 70

next_action:
  实现 Tool Registry

blocked_by: null
```

作用：告诉 AI 项目做到哪里了。

## memory.md

长期记忆。

```markdown
# Long-term Memory

用户：
- 企业开发者

环境：
- 无外部API
- 使用VS Code Copilot

目标：
- 构建企业Agent Harness
```

作用：记录长期不变的信息。

## todo.md

待办事项。

```markdown
# Current Sprint

- [x] Memory Layer
- [ ] Tool Registry
- [ ] Approval Layer

# Next Sprint

- Monitoring
- MCP Integration
```

作用：任务管理。

## decisions.md

决策记录。

```markdown
# Decision Log

## 2026-08-19

决定：
采用 Markdown+YAML 存储Harness状态

原因：
所有AI均支持
```

作用：记录短期决策。

## journal/

每日工作日志。

例如：

```text
journal/
└── 2026-08-19.md
```

内容：

```markdown
完成：
- Memory模块

问题：
- Tool Registry设计待确认

下一步：
- 实现Approval Layer
```

作用：恢复最近上下文。

---

# Architecture Decision Records（ADR）

不要把重要架构知识放进 memory。

推荐：

```text
adr/
```

例如：

```text
ADR-001-search.md
ADR-002-database.md
ADR-003-framework.md
```

示例：

```markdown
# ADR-001

状态：Accepted

决策：采用 Azure AI Search

原因：符合企业标准

替代方案：
- Milvus
- Elasticsearch
```

作用：长期保存架构决策。

---

# AI启动标准流程

无论使用：

```text
Claude Code
VS Code Copilot
Cursor
Gemini
```

统一执行：

```text
请按照 .ai/START.md 初始化项目
```

Agent执行：

```text
START.md
 ↓
rules.md
 ↓
state.yaml
 ↓
todo.md
 ↓
memory.md
 ↓
开始工作
```

形成标准 Bootstrap 流程。

---

# 推荐 Prompt

每次开启新对话：

```text
请作为项目Agent工作。

先执行：
.ai/START.md

然后：
1. 总结项目状态
2. 识别当前任务
3. 给出下一步建议
4. 如有更新同步到：
   - state.yaml
   - todo.md
   - journal
```

---

# 为什么这样设计

## 不依赖任何厂商

可兼容：

```text
Claude
Copilot
Cursor
Gemini
未来的新模型
```

## 不依赖任何 API

即使：

```text
无法访问 OpenAI
无法访问 Azure OpenAI
```

依然可运行。

## 可以纳入 Git

推荐：

```bash
git add .ai/
git add adr/
```

这样：

```text
记忆
决策
状态
任务
```

全部版本化。

---

# 最终目标架构

```text
Project
│
├── Source Code
│
├── Harness
│   ├── Rules
│   ├── State
│   ├── Memory
│   ├── Decisions
│   ├── Journal
│   └── ADR
│
└── AI Clients
    ├── Claude Code
    ├── VS Code Copilot
    ├── Cursor
    └── Gemini
```

核心思想：

> Harness 不属于某个 AI，而属于项目本身。任何 AI 都只是 Harness 的客户端。
>
> 不要让 AI 拥有记忆，要让项目拥有记忆。
```text
.ai/
├── START.md
├── rules.md
├── state.yaml
├── memory.md
├── todo.md
├── decisions.md
├── prompt-template.md
├── workflow.md
└── journal/
