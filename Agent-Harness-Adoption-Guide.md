# Agent Harness 跨项目引入指南

本文总结 Stride 在真实开发任务中迭代 Agent Harness 的经验，并给出一套可复制到其他代码库的实施方案。它不是某个 Agent 产品的使用说明，而是一层位于代码库内部、由人和 Agent 共同遵守的工程协议。

## 1. Harness 要解决什么

Agent 能写代码并不等于任务可以可靠交付。跨会话、脏工作区、外部服务和真机测试会带来几个典型问题：

- 新会话不知道上一会话做到哪里，重复工作或遗漏验证。
- “任务完成”只存在于对话里，代码库中的状态已经过期。
- 自动化测试、真机验证和人工产品验收混为一谈，导致任务长期阻塞。
- 工作区原本已有大量未提交修改，Agent 无法证明哪些文件属于当前任务。
- 数据库迁移已经部署，但仓库里没有可靠记录，下一位 Agent 可能重复执行或错误推断。
- 验证结果只写成自然语言，无法检查任务是否真的满足完成条件。
- 人工 QA 被交接后容易遗忘，后续功能继续叠加在未确认行为之上。

Harness 的目标不是增加文档数量，而是建立下面这条可验证链路：

> 用户意图 → 可持久化任务 → 明确所有权 → 自动化完成证据 → 独立人工 QA → 可恢复的项目状态

## 2. 从 Stride 暴露问题得到的核心经验

### 2.1 当前状态必须只有一个权威来源

早期 Harness 同时在多个文件里描述“当前任务”，容易出现一个地方已完成、另一个地方仍显示进行中。最终约定：

- `.ai/state.yaml.in_flight` 是当前已认领任务的唯一权威列表。
- `.ai/todo.md` 是跨会话 backlog 的权威视图。
- `.ai/tasks/<TASK-ID>.md` 保存任务范围和完成证据。
- Journal 只追加历史，不承担当前状态职责。

相同事实可以在不同文件中以不同用途出现，但必须由验证器检查它们的一致性。

### 2.2 开发完成不能依赖人在场

Supabase 登录任务证明了一个重要边界：Google OAuth、权限弹窗、真实账号和物理设备验证都依赖人。如果把它们放进开发完成门槛，即使代码、测试和构建全部通过，任务也会无限期停留在进行中。

因此必须区分：

- **Development completion**：Agent 可以独立、可重复执行的单元测试、集成测试、构建、静态检查和数据库自动化测试。
- **Post-completion manual acceptance**：真机、真实账号、权限、视觉体验、第三方控制台和其他需要人操作的验证。

开发任务在自动化门槛通过后即可完成。人工验收若发现问题，则创建或重新打开一个具体 Bug，而不是让原任务永久等待。

### 2.3 人工 QA 独立于任务状态，但不能被遗忘

“不阻塞开发完成”不等于“可以忘记”。解决方案是独立的 `.ai/manual-qa.yaml`：

- 已完成开发任务仍可有 `pending` 或 `partial` 的人工验收项。
- 每次认领新的实现任务前，验证器检查是否还有需要确认的项目。
- 有待确认项目时，用户必须选择：
  - `completed`：人工检查已经完成并通过；
  - `keep_pending_and_continue`：保留提醒，但允许本次继续；
  - `defect_found`：登记失败检查，并优先建立或恢复 Bug 任务。

这个门禁不是要求用户每次都完成 QA，而是强制做一次有记录的风险确认。

### 2.4 脏工作区必须做任务归属，而不是假装干净

真实项目经常有用户自己的未提交修改。任务认领时保存一次精确的 `git status --porcelain=v1` 快照，并在任务中声明 `allowed_paths`。完成时就能区分：

- 任务开始前已经存在的修改；
- 当前任务允许触碰的路径；
- 既不在快照里、也不在任务范围内的异常修改。

Harness 不应把“`in_flight` 为空”解释为“工作区干净”，也不应擅自还原不属于当前任务的改动。

### 2.5 外部系统状态需要记录，但仓库不能保存秘密

数据库迁移、云端策略和部署状态不是 Git 本身能够证明的。使用 `.ai/external-state.yaml` 保存：

- 外部系统和项目的非敏感标识；
- 部署产物在仓库中的路径；
- `planned`、`applied`、`verified`、`failed` 或 `unknown` 状态；
- 应用时间、验证时间和证据摘要。

密钥、令牌、第三方 Client Secret、管理员连接串和高权限服务密钥不得写入 Harness。Harness 只记录“配置是否完成”和“如何验证”，不记录秘密本身。

### 2.6 完成声明必须是机器可检查的数据

任务文件使用 YAML front matter 保存：

- 状态、Owner、基线 commit 和时间；
- 允许路径、实际变更路径和 preflight 快照；
- 自动化完成门槛；
- 人工验收状态；
- 每条验证命令、结果、时间和证据。

自然语言正文负责解释目标和约束，结构化 front matter 负责让验证器拒绝自相矛盾的完成声明。

## 3. 推荐目录结构

最小可用结构如下：

```text
AGENTS.md
.ai/
├── START.md
├── rules.md
├── workflow.md
├── state.yaml
├── state.schema.json
├── todo.md
├── prompt-template.md
├── task-frontmatter.schema.json
├── manual-qa.yaml
├── manual-qa.schema.json
├── external-state.yaml
├── external-state.schema.json
├── memory.md
├── decisions.md
├── tasks/
├── journal/
├── adr/
└── snapshots/
Scripts/
├── validate-harness.sh
└── validate-harness.rb
```

各文件职责：

| 文件 | 职责 | 更新方式 |
| --- | --- | --- |
| `AGENTS.md` | 项目事实、目录、构建命令、平台约束，并指向 Harness 入口 | 架构或标准命令变化时更新 |
| `.ai/START.md` | 阅读顺序和文件地图 | Harness 结构变化时更新 |
| `.ai/rules.md` | Agent 行为规则和完成边界 | 流程规则变化时更新 |
| `.ai/workflow.md` | 从 intake 到完成、恢复和人工 QA 的操作步骤 | 工作流变化时更新 |
| `.ai/state.yaml` | 当前阶段、进行中任务、阻塞项、下一步和工作区观察 | 任务开始、阻塞、恢复、结束时更新 |
| `.ai/todo.md` | Todo、In progress、Done backlog | 任务状态变化时移动条目 |
| `.ai/tasks/` | 持久化任务 Brief 与验证证据 | 认领、里程碑、完成时更新 |
| `.ai/manual-qa.yaml` | 已完成开发任务的独立人工验收队列 | 交接、用户反馈和新任务前确认时更新 |
| `.ai/external-state.yaml` | 非敏感的外部部署事实 | 外部操作和验证后更新 |
| `.ai/journal/` | 会话时间线和恢复点 | 只追加，不重写历史 |
| `.ai/decisions.md` | 非平凡决策及原因 | 只追加 |
| `.ai/adr/` | 架构级决策 | 产生长期影响的决策时新增 |
| `.ai/snapshots/` | 任务开始前的工作区快照 | 每次认领实现任务时新增 |
| `Scripts/validate-harness.*` | 跨文件、Schema、证据和门禁校验 | Harness 模型变化时同步更新 |

当前 Stride 实现可作为参考：[入口](../.ai/START.md)、[规则](../.ai/rules.md)、[工作流](../.ai/workflow.md)、[任务模板](../.ai/prompt-template.md)和[验证器](../Scripts/validate-harness.rb)。迁移时应复制设计，而不是保留其中的 Stride 名称、iOS 命令或 Supabase 示例。

## 4. 必须固定的状态模型

### 4.1 Task ID

使用稳定且可搜索的 ID，例如：

- `AUTH-001`
- `SYNC-002`
- `BUG-014`
- `HARNESS-003`

同一个 ID 必须贯穿 backlog、task file、state、journal、manual QA 和提交说明。不要使用“登录功能”“刚才那个问题”作为跨会话标识。

### 4.2 任务状态

推荐任务文件状态：

- `todo`
- `in_progress`
- `blocked`
- `done`

`blocked` 必须有结构化 blocker，包含 Owner、摘要、创建时间和解决状态。困难、测试慢或尚未完成不是 blocker；只有确实需要新权限、用户决策或外部状态变化时才应阻塞。

### 4.3 人工 QA 状态

推荐独立状态：

- `pending`
- `partial`
- `passed`
- `failed`
- `waived`

任务 front matter 的 `manual_acceptance` 应与对应 QA 项一致。没有人工验收需求时使用 `not_required`。

### 4.4 验证证据

每条验证至少包含：

```yaml
- id: unit_tests
  command: "<project test command>"
  result: passed
  verified_at: "2026-08-20T10:00:00+00:00"
  evidence: "42 tests passed; report stored at <artifact path>."
```

证据应该说明“验证了什么”，不能只写“运行成功”。如果命令没有运行，使用 `not_run`、`verified_at: null` 并解释原因；完成任务不得包含失败的必需验证。

## 5. 标准任务生命周期

### 第 0 步：会话启动

固定阅读顺序：

1. `AGENTS.md` 中的项目入口说明；
2. `.ai/START.md`；
3. `.ai/rules.md`；
4. `.ai/state.yaml`；
5. `.ai/todo.md`；
6. `.ai/manual-qa.yaml`；
7. 与当前任务相关的 task file 和最新 journal。

先恢复进行中的任务，再认领新任务。只读审查、解释和诊断通常不需要修改 Harness 状态。

### 第 1 步：新任务前人工 QA 门禁

运行：

```sh
./Scripts/validate-harness.sh --before-new-task
```

建议退出码：

- `0`：Harness 一致，可以认领任务；
- `1`：Harness 本身无效，先修复状态或 Schema 错误；
- `2`：存在待确认人工 QA，先取得用户的三个选项之一。

只有实现类或会改变仓库状态的新任务需要此门禁；纯问答不应制造虚假的任务记录。

### 第 2 步：Intake 与 preflight

为直接请求分配 Task ID，定义目标、范围、自动化完成标准、人工验收、约束和明确的 out-of-scope。然后：

1. 记录当前 branch 和 base commit；
2. 检查 `git status`；
3. 保存 `.ai/snapshots/<TASK-ID>-preflight.txt`；
4. 创建 `.ai/tasks/<TASK-ID>.md`；
5. 将 todo 条目移动到 In progress；
6. 写入 `state.yaml.in_flight`；
7. 在当天 journal 追加 Started 记录；
8. 运行一次普通 Harness 校验。

### 第 3 步：实现与里程碑

只在 `allowed_paths` 内工作；发现必须扩大范围时，先更新 Task Brief 和 state，并说明原因。关键里程碑、决策、验证失败和精确下一步追加到 journal。

不要把过程日志全部塞进 `state.yaml`。State 应保持短小，只保存恢复当前工作的必要信息。

### 第 4 步：自动化完成

完成前至少：

1. 执行 task-specific 单元或集成测试；
2. 执行项目规定的 build、lint、type-check 或静态检查；
3. 验证共享目标、数据迁移或兼容性要求；
4. 记录每条命令和结果；
5. 检查实际变更路径是否在允许范围；
6. 运行 Harness 验证器。

自动化门槛全部通过后：

- Task 状态改为 `done`；
- 从 `state.yaml.in_flight` 移除；
- backlog 移到 Done；
- 更新 `changed_paths`、`completed_at` 和 verification；
- 在 journal 追加 Done 和最终验证；
- 若仍有人工作业，写入 `manual-qa.yaml`。

### 第 5 步：人工验收和缺陷回流

向人交付一个可执行的检查清单，每项只验证一个行为，并说明期望结果。用户反馈后：

- 通过：记录证据和时间；
- 暂缓：保留 pending；
- 失败：记录现象，并创建或恢复具体 Bug；
- 明确不再验证：记录为 waived 及原因。

人工 QA 不应只写在聊天中，因为下一次会话无法可靠恢复。

## 6. 通用 Task Brief 模板

下面是可迁移的最小版本，字段应通过 JSON Schema 校验：

```markdown
---
schema_version: 1
id: FEATURE-001
title: Short task title
status: todo
owner: unassigned
created_at: "YYYY-MM-DDTHH:MM:SS+00:00"
completed_at: null
completion_gate: automated
manual_acceptance: pending
base_commit: 0000000
allowed_paths:
  - src/feature/
changed_paths: []
preflight_snapshot: null
verification: []
---

# FEATURE-001: Short task title

## Goal

一句话描述完成后的真实结果。

## Context

记录架构背景、依赖、外部前置条件和相关历史。

## Development completion criteria

- Agent 能独立执行并证明的测试、构建和静态检查。

## Post-completion manual acceptance

- 需要真机、真实账号、权限或人工体验判断的检查。

## Constraints

- 允许和禁止的技术或文件范围。

## Out of scope

- 本任务明确不处理的内容。

## Verification

- `<test command>`
- `<build or lint command>`
```

关键规则是：Development completion criteria 中不能混入只有人才能执行的步骤。

## 7. 人工 QA 队列模板

```yaml
schema_version: 1
prompt_before_new_task: true
confirmation_options:
  - completed
  - keep_pending_and_continue
  - defect_found
items:
  - task_id: "FEATURE-001"
    title: "Human-readable feature title"
    status: "partial"
    confirmation_required: true
    created_at: "YYYY-MM-DDTHH:MM:SS+00:00"
    updated_at: "YYYY-MM-DDTHH:MM:SS+00:00"
    completed_at: null
    last_confirmed_at: null
    last_confirmation_outcome: null
    checklist:
      - id: "physical_device_flow"
        title: "Flow succeeds on a physical device"
        status: "pending"
        evidence: ""
    notes: "Ask before the next implementation task."
```

当用户选择 `keep_pending_and_continue` 时，必须更新时间和 outcome，但不能把 checklist 假装标记为通过。这样既允许继续开发，也保留下一次提醒。

## 8. 验证器应该检查什么

一个实用验证器至少应覆盖：

1. YAML 和 JSON 是否可解析；
2. State、Task、Manual QA 和 External State 是否满足各自 Schema；
3. Task ID 是否唯一且格式统一；
4. task file 状态、todo 分区和 `state.in_flight` 是否一致；
5. 进行中任务是否存在 journal 和 preflight snapshot；
6. done 任务是否有完成时间、实际变更路径和至少一条通过证据；
7. `changed_paths` 是否落在 `allowed_paths` 内；
8. Task Brief 是否同时包含自动化完成和人工验收章节；
9. Manual QA 状态是否与 Task front matter 一致；
10. 外部部署记录指向的仓库产物是否存在；
11. Harness 文档中的相对链接是否有效；
12. 外部状态和 Harness 文件是否疑似包含秘密；
13. 当前脏路径是否属于任务范围或已经存在于 preflight 快照；
14. `--before-new-task` 模式是否为未完成 QA 返回独立退出码。

验证器自身要满足两个条件：依赖少、执行快。这样 Agent 才会在 intake 和 completion 两个阶段都实际运行它。Stride 使用 shell wrapper 加标准库 Ruby；其他项目可以根据团队环境换成 Python、Node.js、Go 或现有构建系统，但应保留退出码语义。

## 9. 中断恢复协议

Agent 或会话被中断后，恢复顺序必须固定：

1. 从 `state.yaml.in_flight` 找到任务和 Owner；
2. 阅读对应 Task Brief；
3. 阅读该任务最新 journal 记录；
4. 比较当前 branch、worktree 与 preflight 快照；
5. 判断当前 Agent 是否有所有权或已得到明确 handoff；
6. 从 journal 记录的“精确下一步”继续，而不是重新执行全部工作。

中断前应尽量写下：已经完成的里程碑、变更文件、最后一次验证、当前 blocker 和下一条命令。恢复协议的价值取决于这些信息是否足够具体。

## 10. 在另一个项目中的渐进式引入方案

不要一次复制全部文件再补内容。推荐按下面顺序引入，每一步都可独立验收。

### 阶段 A：建立入口和项目事实

- 新建或整理 `AGENTS.md`，只写稳定的项目事实：目录地图、构建/测试命令、平台约束和不可破坏的兼容性规则。
- 创建 `.ai/START.md` 和 `.ai/rules.md`，规定所有 Agent 的固定阅读顺序。
- 明确哪些内容属于项目事实，哪些属于工作纪律，避免两个文件互相覆盖。

验收：新会话只读这几个文件就能找到代码入口并执行正确的基础验证。

### 阶段 B：建立最小任务状态

- 添加 `state.yaml`、`todo.md`、`tasks/`、`journal/` 和 `snapshots/`。
- 选一个真实但范围较小的功能作为首个端到端任务。
- 记录 Task ID、Owner、base commit、allowed paths 和 preflight snapshot。

验收：中断会话后，另一会话能不依赖聊天历史恢复到精确下一步。

### 阶段 C：Schema 和自动校验

- 为 state 和 task front matter 添加 JSON Schema。
- 编写一条 `validate-harness` 命令，先检查解析、必填字段和 task/todo/state 一致性。
- 把该命令放入项目的日常验证或 CI。

验收：故意制造状态不一致时，验证器能稳定失败并指出文件和原因。

### 阶段 D：拆分人工验收

- 在 Task Brief 中拆分自动化完成与人工验收。
- 增加独立 `manual-qa.yaml` 和 pre-new-task 门禁。
- 规定三个确认结果以及 `defect_found` 的 Bug 回流机制。

验收：一个依赖真机的任务可以在自动化验证通过后正常完成，同时下一个实现任务开始前仍会提醒未做的真机 QA。

### 阶段 E：外部状态和安全边界

- 对数据库、云部署、Feature Flag 或第三方配置增加非敏感 external state。
- 为外部操作保存 artifact、时间、状态和证据。
- 在验证器中加入秘密模式扫描，并在规则中明确禁止项。

验收：新会话能判断迁移是否已经部署，但无法从 Harness 读取任何秘密。

### 阶段 F：真实任务复盘

- 用真实任务而不是虚构示例验证 Harness。
- 完成后检查：哪里仍依赖聊天记忆、哪里重复记录、哪个字段没人维护、哪条规则无法自动验证。
- 删除无价值字段，给关键不变量增加验证，不要让 Harness 只增长不收敛。

## 11. 需要按项目定制的部分

迁移时必须替换：

- 项目名称、语言、最低平台版本和源码目录；
- 标准测试、构建、lint、format 和部署命令；
- 用户界面字符串、数据库迁移和共享模块的专项检查；
- 设备、浏览器、权限或真实账号的人工验收类型；
- CI 环境中可用的解释器；
- Task ID 分类和团队 Owner 命名约定；
- 哪些外部系统需要 external state；
- 项目自己的秘密模式和禁止入库规则。

不应照搬：

- Stride 的 iOS 目录和 `xcodebuild` 命令；
- Supabase 项目引用、OAuth 回调或数据库表名；
- 当前代码库的任务历史、journal、snapshot 和 QA 记录；
- 任何用户、设备或环境专属标识。

## 12. 常见失败方式

### Harness 只是说明书

如果没有验证器，规则最终会与状态脱节。关键不变量必须尽量机器化，例如 done 任务必须有证据、in-flight 必须与 todo 对齐。

### 记录过多的即时细节

State 和 memory 不是完整聊天存档。只保存下一会话恢复和做决策所需的稳定事实；过程细节进入 append-only journal。

### 每个小问答都创建任务

这会产生大量无价值状态维护。只有实现类、会改变仓库、需要多步验证或可能跨会话的工作才需要认领任务。

### 完成条件写成“用户觉得没问题”

这无法重复验证。开发完成应以自动化命令和具体行为为准；人的感受和真机流程单列到 manual acceptance。

### 人工 QA 重新阻塞已完成开发

待验收不等于代码还在开发。保持开发任务 done，把风险放在独立 QA 队列；发现缺陷时再建立 Bug。

### 把秘密当成外部状态保存

Harness 需要知道“配置完成”，不需要知道 Secret 的值。秘密必须留在受控环境变量、Secret Manager、钥匙串或服务控制台中。

### 在脏工作区里扩大所有权

Agent 不应因为用户要求实现一个功能，就默认拥有所有未提交文件。preflight snapshot 和 allowed paths 是最低限度的保护。

### 规则无法执行

例如要求“每次都完整真机构建”，但 CI 和 Agent 环境没有设备。这种规则应拆成可自动完成的 build/test 与单独的人工设备交接，否则团队会被迫忽略规则。

## 13. Harness 自身的验收清单

引入完成后，至少进行一次故障注入测试：

- [ ] 将 task 标为 done 但删除 verification，验证器应失败。
- [ ] 在 todo 中保留 In progress、从 state 移除任务，验证器应失败。
- [ ] 添加超出 allowed paths 的 changed path，验证器应失败。
- [ ] 指向不存在的 snapshot、journal 或 external artifact，验证器应失败。
- [ ] 创建 pending manual QA，普通校验应通过，pre-task 校验应返回专用退出码。
- [ ] 选择 `keep_pending_and_continue` 后，QA 仍应保持 pending，并记录确认时间。
- [ ] 模拟 `defect_found`，应先产生具体 Bug 再允许无关开发。
- [ ] 在 external state 中放入测试用秘密模式，验证器应拒绝。
- [ ] 中断一个任务，新会话应能从 state、task 和 journal 恢复。
- [ ] 在一个原本就很脏的工作区认领任务，不应误归属或覆盖用户改动。

## 14. 最小落地标准

如果时间有限，先保证以下六件事：

1. 有统一入口和固定阅读顺序；
2. `state.in_flight` 是唯一当前任务来源；
3. 每个实现任务有 Task ID、范围、base commit 和 preflight snapshot；
4. 自动化开发完成与人工验收严格分离；
5. 未完成人工 QA 在下一实现任务前必须被明确确认；
6. 一条快速命令能检查跨文件一致性并拒绝无证据的完成声明。

这六项已经能显著降低跨会话遗忘、任务假完成、真机阻塞、外部状态丢失和脏工作区误操作。其余 memory、ADR、external state 和更严格 Schema 可以随着真实问题逐步加入。

## 15. 最后的设计原则

好的 Harness 应该让 Agent 更自主，而不是更机械。实现这一点需要保持三个平衡：

- **规则少但可执行**：能自动检查的就不要只写成提醒。
- **状态短但可恢复**：当前状态保存下一步，历史细节进入 journal。
- **自动化决定完成，人类决定接受**：两者都重要，但生命周期不同。

最有效的引入方式，是把 Harness 当作代码来维护：有 Schema、有测试、有真实任务验证，也允许根据实际摩擦持续删改。不要追求一次设计完美；先让下一位 Agent 能够安全、准确地接手当前工作。
