# Claude Code Agents（专用智能体）系统深度分析

> 分析日期：2026-04-08
> 分析范围：`/Volumes/Data/agent-frameworks/claude-code/plugins/` 下所有 `agents/` 目录

---

## 目录

1. [Agent 完整清单与对比表格](#1-agent-完整清单与对比表格)
2. [Agent Frontmatter 格式详解](#2-agent-frontmatter-格式详解)
3. [工具权限控制分析](#3-工具权限控制分析)
4. [Agent 触发条件设计](#4-agent-触发条件设计)
5. [多 Agent 并行编排](#5-多-agent-并行编排)
6. [Agent 系统提示设计质量分析](#6-agent-系统提示设计质量分析)
7. [关键发现总结](#7-关键发现总结)

---

## 1. Agent 完整清单与对比表格

共发现 **15 个 Agent**，分布在 5 个插件中：

| Agent 名称 | 插件 | model | color | tools 数量 | 触发场景 |
|---|---|---|---|---|---|
| `code-explorer` | feature-dev | sonnet | yellow | 10 | 代码探索、追踪特性实现 |
| `code-architect` | feature-dev | sonnet | green | 10 | 功能架构设计、蓝图生成 |
| `code-reviewer` | feature-dev | sonnet | red | 10 | 代码审查（功能开发流程内） |
| `conversation-analyzer` | hookify | inherit | yellow | 2 | 分析对话记录寻找 hook 触发模式 |
| `agent-creator` | plugin-dev | sonnet | magenta | 2 | 创建新 Agent 配置文件 |
| `plugin-validator` | plugin-dev | inherit | yellow | 4 | 验证插件结构与 manifest |
| `skill-reviewer` | plugin-dev | inherit | cyan | 3 | 审查 Skill 质量与描述有效性 |
| `code-reviewer` | pr-review-toolkit | **opus** | green | 未限制 | PR 代码审查（最高质量需求） |
| `code-simplifier` | pr-review-toolkit | **opus** | 未指定 | 未限制 | 代码简化与可读性提升 |
| `comment-analyzer` | pr-review-toolkit | inherit | green | 未限制 | 代码注释准确性分析 |
| `pr-test-analyzer` | pr-review-toolkit | inherit | cyan | 未限制 | PR 测试覆盖率分析 |
| `silent-failure-hunter` | pr-review-toolkit | inherit | yellow | 未限制 | 静默失败与错误处理审查 |
| `type-design-analyzer` | pr-review-toolkit | inherit | pink | 未限制 | 类型设计与不变量分析 |
| `agent-sdk-verifier-py` | agent-sdk-dev | sonnet | 未指定 | 未限制 | Python Agent SDK 应用验证 |
| `agent-sdk-verifier-ts` | agent-sdk-dev | sonnet | 未指定 | 未限制 | TypeScript Agent SDK 应用验证 |

---

## 2. Agent Frontmatter 格式详解

### 2.1 标准 Frontmatter 字段

每个 Agent 的 `.md` 文件以 YAML frontmatter 开头，主要字段如下：

```yaml
---
name: code-explorer
description: Deeply analyzes existing codebase features by tracing execution paths...
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
model: sonnet
color: yellow
---
```

**字段说明：**

| 字段 | 是否必须 | 类型 | 说明 |
|---|---|---|---|
| `name` | 必须 | string | Agent 标识符，小写 + 连字符，3-50 字符 |
| `description` | 必须 | string | 触发描述，包含 `<example>` 块 |
| `model` | 必须 | enum | 模型选择：`inherit` / `sonnet` / `opus` |
| `color` | 可选 | enum | UI 颜色标识 |
| `tools` | 可选 | list/array | 工具白名单，缺省则获得全部权限 |

### 2.2 model 选择策略

系统采用三级模型选择策略：

**`inherit`（继承父会话）**
- 适用场景：轻量验证、分析类任务
- 代表 Agent：`conversation-analyzer`、`plugin-validator`、`skill-reviewer`、`comment-analyzer`、`pr-test-analyzer`、`silent-failure-hunter`、`type-design-analyzer`
- 优点：成本最低，利用父会话已选择的模型

**`sonnet`（固定中端模型）**
- 适用场景：需要较强推理能力的深度分析与创作
- 代表 Agent：`code-explorer`、`code-architect`、`code-reviewer`（feature-dev）、`agent-creator`、`agent-sdk-verifier-py/ts`
- 优点：平衡质量与速度，适合并行批量调用

**`opus`（固定高端模型）**
- 适用场景：最高质量要求的任务（PR 级别代码审查、代码简化）
- 代表 Agent：`code-reviewer`（pr-review-toolkit）、`code-simplifier`（pr-review-toolkit）
- 优点：最强推理能力，适合需要细致判断的任务

**关键决策逻辑**（来自 `agent-creator` 系统提示）：

```markdown
- **Model**: Use `inherit` unless user specifies (sonnet for complex, haiku for simple)
- sonnet for complex, haiku for simple（最小化模型）
- opus 专用于需要最高精度的审查场景
```

### 2.3 color 字段的含义与 UI 用途

Color 字段用于在 Claude Code UI 中区分不同类型的 Agent，形成语义化颜色编码：

| 颜色 | 语义 | 代表 Agent |
|---|---|---|
| `yellow` | 验证、警告、谨慎型 | `code-explorer`、`conversation-analyzer`、`plugin-validator`、`silent-failure-hunter` |
| `green` | 生成、创建、通过型 | `code-architect`、`code-reviewer`（pr-review-toolkit）、`comment-analyzer` |
| `red` | 安全、严重问题、阻断型 | `code-reviewer`（feature-dev） |
| `cyan` | 分析、信息查询型 | `skill-reviewer`、`pr-test-analyzer` |
| `magenta` | 创意转化型 | `agent-creator` |
| `pink` | 专项分析型 | `type-design-analyzer` |

> **注意**：`code-simplifier`、`agent-sdk-verifier-py/ts` 未指定颜色，使用系统默认色。

---

## 3. 工具权限控制分析

### 3.1 工具权限分级对比

系统体现了严格的最小权限原则（Principle of Least Privilege）：

**最严格限制（2个工具）**

```yaml
# conversation-analyzer
tools: ["Read", "Grep"]

# agent-creator
tools: ["Write", "Read"]
```

- `conversation-analyzer`：只需读对话记录 → 仅 Read + Grep
- `agent-creator`：只需读取上下文并写入新文件 → 仅 Write + Read

**中等限制（3-4个工具）**

```yaml
# skill-reviewer
tools: ["Read", "Grep", "Glob"]

# plugin-validator
tools: ["Read", "Grep", "Glob", "Bash"]
```

- `skill-reviewer`：需要遍历目录、搜索文件 → 增加 Glob
- `plugin-validator`：需要运行验证脚本（如 `jq`）→ 增加 Bash

**宽泛权限（10个工具）**

```yaml
# code-explorer / code-architect / code-reviewer (feature-dev)
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
```

这三个 Agent 获得相同的工具集，包含：
- 文件系统读取：`Glob, Grep, LS, Read, NotebookRead`
- 网络访问：`WebFetch, WebSearch`
- 任务管理：`TodoWrite`
- Shell 控制：`KillShell, BashOutput`
- **注意**：无 `Write/Edit` 权限 → 只读分析，不直接修改代码

**完全权限（未指定）**

`pr-review-toolkit` 所有 Agent 和 `agent-sdk-verifier-*` 均未指定 tools，可使用所有工具。

### 3.2 工具组合的语义

| 工具组合 | 语义 | 适用场景 |
|---|---|---|
| `Read + Grep` | 只读文本分析 | 对话日志分析 |
| `Read + Write` | 读取+创作 | 生成新文件（Agent/Skill） |
| `Read + Grep + Glob` | 只读目录遍历 | 结构验证、Skill 审查 |
| `Read + Grep + Glob + Bash` | 只读+脚本执行 | 插件验证（JSON解析等） |
| `Glob+Grep+LS+Read+WebFetch+...` | 深度只读探索 | 代码理解、架构分析 |
| 未指定（全工具） | 完全自主 | 复杂审查与修改任务 |

### 3.3 关键设计洞察

**写权限的谨慎使用**：`code-explorer`、`code-architect`、`code-reviewer` 虽然有 10 个工具，但均**不包含** `Write`、`Edit`、`MultiEdit` 等写权限工具，确保这些分析 Agent 是纯只读的。唯一明确包含 `Write` 的 Agent 是 `agent-creator`，且同时限制了其他工具。

---

## 4. Agent 触发条件设计

### 4.1 description 字段结构分析

`description` 字段是 Agent 触发的核心机制，有两种设计模式：

**模式一：纯文字描述（feature-dev 风格）**

```markdown
description: Deeply analyzes existing codebase features by tracing execution paths,
mapping architecture layers, understanding patterns and abstractions, and documenting
dependencies to inform new development
```

- 无 `<example>` 块
- 简洁的功能描述
- 依赖父命令（feature-dev.md）显式指定何时启动

**模式二：描述 + `<example>` + `<commentary>` 结构（plugin-dev / pr-review-toolkit 风格）**

```markdown
description: Use this agent when the user asks to "create an agent", "generate an agent",
"build a new agent"... Trigger when user wants to create autonomous agents for plugins.

<example>
Context: User wants to create a code review agent
user: "Create an agent that reviews code for quality issues"
assistant: "I'll use the agent-creator agent to generate the agent configuration."
<commentary>
User requesting new agent creation, trigger agent-creator to generate it.
</commentary>
</example>
```

**`<example>` 块的三层结构：**

1. `Context`：触发场景的背景描述（几词短语）
2. `user`：用户具体说的话（示例输入）
3. `assistant`（可选）：助手响应模式
4. `<commentary>`：为什么应该触发此 Agent 的解释

### 4.2 精确触发设计——避免误触发策略

**策略一：限定关键词短语**（`agent-creator`）

```markdown
the user asks to "create an agent", "generate an agent", "build a new agent",
"make me an agent that..."
```
通过列举精确短语而非宽泛描述，降低误触发风险。

**策略二：否定条件界定**（`code-simplifier`）

```markdown
This agent should be triggered automatically after completing a coding task
or writing a logical chunk of code.
```
描述触发时机（完成编码后），暗示不在编码中途触发。

**策略三：主动触发 vs 显式触发区分**（`plugin-validator`）

```markdown
Also trigger proactively after user creates or modifies plugin components.
```
区分"用户显式请求"与"系统自动触发"两种触发模式。

**策略四：多例子覆盖不同语境**（`skill-reviewer`）

三个 `<example>` 分别覆盖：
1. 用户新建 Skill（主动触发）
2. 用户明确请求审查（显式触发）
3. 用户修改了 Skill 描述（变更触发）

### 4.3 不同 Agent 触发精确度对比

| Agent | 触发模式 | 精确度 | 特点 |
|---|---|---|---|
| `code-explorer` | 父命令显式调用 | 极高 | 不独立触发，由 feature-dev 命令编排 |
| `code-architect` | 父命令显式调用 | 极高 | 同上 |
| `conversation-analyzer` | 关键词 + 场景描述 | 高 | 有 `<example>` 覆盖两种场景 |
| `agent-creator` | 多精确关键词 | 高 | 3个 `<example>` 区分不同语境 |
| `plugin-validator` | 显式 + 主动双模式 | 中高 | 3个 `<example>` 含主动触发 |
| `skill-reviewer` | 显式 + 主动双模式 | 中高 | 3个 `<example>` 含变更触发 |
| `code-reviewer`（PR） | 场景描述 + 3例子 | 中 | 覆盖"完成功能"/"写代码后"/"创建PR前" |
| `code-simplifier` | 场景描述 + 3例子 | 中 | 自动触发为主，描述触发时机 |
| `silent-failure-hunter` | 场景描述 + 3例子 | 中 | 错误处理相关代码变更时触发 |
| `type-design-analyzer` | 场景描述 + 2例子 | 中 | 新增类型时触发 |
| `pr-test-analyzer` | 场景描述 + 3例子 | 中 | PR 相关上下文触发 |
| `comment-analyzer` | 场景描述 + 3例子 | 中 | 注释相关操作触发 |

---

## 5. 多 Agent 并行编排

### 5.1 feature-dev 命令：三阶段并行 Agent 编排

`feature-dev.md` 命令文件定义了一个复杂的多阶段 Agent 编排流程：

```
feature-dev 命令
│
├── Phase 2: Codebase Exploration（代码探索阶段）
│   ├── [并行] code-explorer #1 → "追踪类似特性实现"
│   ├── [并行] code-explorer #2 → "映射架构与抽象层"
│   └── [并行] code-explorer #3 → "分析 UI/测试/扩展点"
│           │
│           └── 汇总：主流程读取所有 Agent 推荐的关键文件
│
├── Phase 4: Architecture Design（架构设计阶段）
│   ├── [并行] code-architect #1 → 最小化变更方案
│   ├── [并行] code-architect #2 → 干净架构方案
│   └── [并行] code-architect #3 → 实用平衡方案
│           │
│           └── 汇总：对比三方案，向用户呈现权衡分析
│
└── Phase 6: Quality Review（质量审查阶段）
    ├── [并行] code-reviewer #1 → 简洁性/DRY/优雅性
    ├── [并行] code-reviewer #2 → Bug/功能正确性
    └── [并行] code-reviewer #3 → 项目规范/抽象层
            │
            └── 汇总：合并最高严重度问题，向用户呈现
```

**关键设计特征：**

1. **同一 Agent 多实例**：相同的 `code-explorer`/`code-architect`/`code-reviewer` Agent 以不同的 prompt 并行运行 2-3 次，每次聚焦不同维度。

2. **Agent 返回文件列表**：命令明确要求 Agent 返回"5-10 个关键文件"，主流程再读取这些文件以建立上下文。

3. **用户确认门禁**：每个并行阶段结束后都有用户确认步骤（`Wait for user answers`、`Ask user which approach they prefer`），防止失控执行。

### 5.2 pr-review-toolkit 命令：专项 Agent 分工机制

```
/review-pr 命令
│
├── Step 1: 分析变更范围（git diff --name-only）
│
├── Step 2: 智能选择 Agent（按变更内容）
│   ├── 总是运行: code-reviewer（通用代码质量）
│   ├── 若有测试文件变更: pr-test-analyzer
│   ├── 若有注释/文档变更: comment-analyzer
│   ├── 若有错误处理变更: silent-failure-hunter
│   ├── 若有新增/修改类型: type-design-analyzer
│   └── 通过基础审查后: code-simplifier（最后润色）
│
├── Step 3: 执行策略选择
│   ├── 顺序模式（默认）: 逐个 Agent 运行，易于跟进
│   └── 并行模式（用户可选）: 同时启动所有 Agent
│
└── Step 4: 结果聚合
    ├── Critical Issues（必须修复）
    ├── Important Issues（应该修复）
    ├── Suggestions（建议）
    └── Positive Observations（亮点）
```

**专项分工的语义边界：**

| Agent | 专项领域 | 不负责 |
|---|---|---|
| `code-reviewer` | 逻辑错误、规范符合性 | 注释、测试覆盖 |
| `pr-test-analyzer` | 测试覆盖质量 | 代码实现 |
| `comment-analyzer` | 注释准确性 | 代码逻辑 |
| `silent-failure-hunter` | 错误处理模式 | 业务逻辑 |
| `type-design-analyzer` | 类型不变量 | 运行时行为 |
| `code-simplifier` | 可读性优化 | 功能验证（最后运行） |

### 5.3 Agent 结果回传主流程机制

Agent 的输出通过以下方式回传：

1. **文本响应**：Agent 以结构化 Markdown 格式返回分析结果（Critical Issues / Warnings / Recommendations 等）
2. **文件列表推荐**：`code-explorer` 的输出包含"绝对关键文件"列表，主流程读取这些文件
3. **评分量化**：`type-design-analyzer` 返回 1-10 数字评分；`code-reviewer` 返回 0-100 置信度；`pr-test-analyzer` 返回 1-10 优先级
4. **主流程聚合**：命令文件的主提示词负责汇总所有 Agent 输出并向用户呈现

---

## 6. Agent 系统提示设计质量分析

### 6.1 三段式黄金结构

高质量 Agent 系统提示普遍遵循 **Core Mission → Analysis Approach → Output Guidance** 三段式结构：

**以 `code-explorer` 为例（最规范结构）**：

```markdown
## Core Mission
Provide a complete understanding of how a specific feature works...

## Analysis Approach
**1. Feature Discovery** ...
**2. Code Flow Tracing** ...
**3. Architecture Analysis** ...
**4. Implementation Details** ...

## Output Guidance
Provide a comprehensive analysis that helps developers understand...
```

**以 `code-reviewer`（feature-dev）为例（另一标准写法）**：

```markdown
## Review Scope        ← 明确作用域
## Core Review Responsibilities  ← 分析方法
## Confidence Scoring  ← 量化机制（独特设计）
## Output Guidance     ← 输出要求
```

### 6.2 各 Agent 系统提示结构对比

| Agent | 结构模式 | 字数估计 | 特色设计 |
|---|---|---|---|
| `code-explorer` | Core Mission → Approach → Output | ~400词 | 4步分析框架，强调文件引用 |
| `code-architect` | Core Process → Output Guidance | ~350词 | 要求"果断决策"，禁止呈现多选项 |
| `code-reviewer`（feature-dev） | Review Scope → Responsibilities → Confidence → Output | ~450词 | **信心评分系统（0-100）** |
| `conversation-analyzer` | Responsibilities → Analysis Process → Output | ~600词 | 结构化输出模板，含边缘案例处理 |
| `agent-creator` | 角色定义 → 6步创建流程 → 质量标准 → 输出格式 | ~1200词 | 最长最详细，包含完整的 Agent 创建元指南 |
| `plugin-validator` | Responsibilities → 10步验证流程 → Output | ~1000词 | 分门别类的验证清单（命令/Agent/Skill/Hook/MCP） |
| `skill-reviewer` | Responsibilities → 8步审查流程 → Output | ~800词 | 含质量量化标准（字数范围、描述长度限制） |
| `code-reviewer`（pr-toolkit） | Review Scope → Responsibilities → Confidence → Output | ~400词 | 与 feature-dev 版高度相似但 model=opus |
| `code-simplifier` | 5条核心原则 → 精化流程 | ~450词 | 明确列出"不要做"的事项 |
| `comment-analyzer` | 使命陈述 → 5维分析 → 输出结构 | ~500词 | "guardian against technical debt"角色定位 |
| `pr-test-analyzer` | Responsibilities → 分析流程 → 评级指南 → 输出 | ~500词 | 优先级 1-10 评分体系 |
| `silent-failure-hunter` | Core Principles → 5步审查流程 → 输出格式 | ~700词 | 最严格的非妥协立场，5条不可协商原则 |
| `type-design-analyzer` | Core Mission → 5维分析框架 → 输出 → 原则 → 反模式 | ~600词 | 四维量化评分（1-10 × 4维度） |
| `agent-sdk-verifier-py` | 验证重点 → 8项检查 → 不关注什么 → 流程 → 报告格式 | ~600词 | 明确声明"What NOT to Focus On" |
| `agent-sdk-verifier-ts` | 同上（TS版） | ~650词 | 增加 TypeScript 特有的编译检查 |

### 6.3 高质量系统提示的共同特征

**特征一：明确的角色定位（Identity Statement）**

每个高质量 Agent 都以强角色宣言开场：
- `"You are an expert code analyst specializing in..."`
- `"You are an elite error handling auditor with zero tolerance for..."`
- `"You are a meticulous code comment analyzer with deep expertise in..."`

**特征二：量化机制的引入**

避免模糊判断，引入具体数值标准：
- 信心评分：0-100（code-reviewer）
- 优先级评分：1-10（pr-test-analyzer）
- 四维评分：1-10 × 4（type-design-analyzer）
- 报告阈值：≥80才上报（code-reviewer）

**特征三：结构化输出模板**

高质量 Agent 均提供精确的输出格式模板，减少格式歧义：

```markdown
## Plugin Validation Report
### Summary
### Critical Issues ([count])
### Warnings ([count])
### Component Summary
### Positive Findings
### Overall Assessment
```

**特征四：显式边缘案例处理**

成熟的 Agent 专门列出边缘情况处理策略：
- `conversation-analyzer`：处理"用户讨论假设性情况"vs"实际问题"
- `plugin-validator`：处理"空目录"/"未知字段"/"文件损坏"
- `agent-sdk-verifier-*`：明确列出"What NOT to Focus On"

**特征五：系统提示复杂度与任务复杂度正相关**

| 系统提示复杂度 | Agent 示例 | 任务特征 |
|---|---|---|
| 简单（~350词） | code-architect | 输出类型单一（架构蓝图） |
| 中等（~500词） | code-reviewer、comment-analyzer | 多维度分析，标准化输出 |
| 复杂（~800-1200词） | agent-creator、plugin-validator | 元级任务（创建/验证其他组件） |

---

## 7. 关键发现总结

### 发现一：双轨触发设计
系统存在两类 Agent：
1. **显式编排型**（feature-dev 的 code-explorer/architect/reviewer）：由命令文件精确控制何时启动、启动多少个、用什么 prompt，Agent 本身无触发描述。
2. **自主触发型**（plugin-dev、pr-review-toolkit 的 Agent）：通过 `description` 中的 `<example>` 块自动判断何时触发，可独立使用也可被命令编排。

### 发现二：最小权限原则的严格执行
工具权限的设计严格遵循最小化原则：
- 最小权限组（2工具）：仅读/写功能
- 中等权限组（3-4工具）：读 + 搜索 + 必要的执行
- 分析专用组（10工具，无写权限）：深度只读探索
- 完全权限（未指定）：复杂审查修改任务

没有任何只读分析 Agent 被授予写入权限——这是整个系统中最重要的安全设计决策。

### 发现三：并行 Agent 编排是核心生产力放大器
`feature-dev` 命令展示了最精妙的编排模式：同一个 Agent 以不同 prompt 并行运行多次，覆盖不同维度。2-3个 `code-explorer` 并行 + 2-3个 `code-architect` 并行 + 2-3个 `code-reviewer` 并行 = 一次 `/feature-dev` 命令可能启动 **6-9个 Agent 并行工作**，极大缩短端到端开发时间。

### 发现四：量化评分机制提升审查可操作性
`code-reviewer` 的信心评分系统（0-100，仅上报 ≥80 的问题）是过滤噪音的关键设计。通过量化阈值而非主观判断决定是否上报，使审查结果更加一致、可预期，且自动过滤低价值噪音。

### 发现五：model=opus 专用于最高质量门禁
仅有 2 个 Agent（`code-reviewer` 和 `code-simplifier`，均来自 pr-review-toolkit）使用 `opus` 模型。这是有意的：这两个 Agent 是代码进入 PR 前的最终质量门禁，使用最强模型确保审查深度，同时通过将它们放在最后阶段控制成本。

### 发现六：元级 Agent 设计——Agent 创造 Agent
`agent-creator` 是系统中最独特的设计：**一个 Agent 的职责是创建其他 Agent**。它的系统提示包含了创建高质量 Agent 的完整方法论，包括 color 选择语义、model 选择逻辑、trigger example 写法等，实现了 Agent 系统的自我复制与扩展能力。

### 发现七：`inherit` 模型是轻量任务的首选
7个 Agent 使用 `inherit` 模型，占比 47%。这表明系统设计者有意识地避免为所有 Agent 都指定固定模型，通过继承父会话模型来降低总体成本，并让用户通过选择父会话模型来统一控制整体质量级别。

### 发现八：PR Review Toolkit 体现"关注点分离"的最佳实践
6个专项审查 Agent 的分工展示了软件工程"关注点分离"原则在 Agent 设计中的应用：每个 Agent 只关注一个维度（注释/测试/错误处理/类型/代码质量/可读性），避免了"什么都管"但"每项都浅"的问题，通过汇总协调层实现全面覆盖。
