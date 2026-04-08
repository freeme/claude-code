# Claude Code Commands（自定义斜杠命令）系统深度分析

> 分析日期：2026-04-08  
> 分析范围：`plugins/*/commands/` 目录及 `.claude/commands/` 目录共 18 个命令文件

---

## 目录

1. [命令系统概览](#1-命令系统概览)
2. [Command Frontmatter 规范](#2-command-frontmatter-规范)
3. [多阶段工作流命令设计](#3-多阶段工作流命令设计)
4. [交互式命令设计模式](#4-交互式命令设计模式)
5. [项目级命令与全局命令](#5-项目级命令与全局命令)
6. [并行 Agent 启动模式](#6-并行-agent-启动模式)
7. [关键发现总结](#7-关键发现总结)

---

## 1. 命令系统概览

Claude Code 的命令系统通过 Markdown 文件实现，每个 `.md` 文件对应一条斜杠命令（`/command-name`）。命令的位置决定其作用域：

| 位置 | 命令前缀 | 作用域 |
|------|---------|--------|
| `plugins/<plugin-name>/commands/<cmd>.md` | `/<plugin-name>:<cmd>` | 插件级（随插件安装） |
| `.claude/commands/<cmd>.md` | `/<cmd>` | 项目级（当前项目可用） |

### 已发现命令清单（共 18 个）

| 命令文件路径 | 斜杠命令 | 所属插件 |
|-------------|---------|---------|
| `plugins/feature-dev/commands/feature-dev.md` | `/feature-dev` | feature-dev |
| `plugins/plugin-dev/commands/create-plugin.md` | `/plugin-dev:create-plugin` | plugin-dev |
| `plugins/hookify/commands/hookify.md` | `/hookify` | hookify |
| `plugins/hookify/commands/configure.md` | `/hookify:configure` | hookify |
| `plugins/hookify/commands/list.md` | `/hookify:list` | hookify |
| `plugins/hookify/commands/help.md` | `/hookify:help` | hookify |
| `plugins/commit-commands/commands/commit.md` | `/commit-commands:commit` | commit-commands |
| `plugins/commit-commands/commands/commit-push-pr.md` | `/commit-commands:commit-push-pr` | commit-commands |
| `plugins/commit-commands/commands/clean_gone.md` | `/commit-commands:clean_gone` | commit-commands |
| `plugins/pr-review-toolkit/commands/review-pr.md` | `/pr-review-toolkit:review-pr` | pr-review-toolkit |
| `plugins/code-review/commands/code-review.md` | `/code-review:code-review` | code-review |
| `plugins/ralph-wiggum/commands/ralph-loop.md` | `/ralph-wiggum:ralph-loop` | ralph-wiggum |
| `plugins/ralph-wiggum/commands/cancel-ralph.md` | `/ralph-wiggum:cancel-ralph` | ralph-wiggum |
| `plugins/ralph-wiggum/commands/help.md` | `/ralph-wiggum:help` | ralph-wiggum |
| `plugins/agent-sdk-dev/commands/new-sdk-app.md` | `/agent-sdk-dev:new-sdk-app` | agent-sdk-dev |
| `.claude/commands/triage-issue.md` | `/triage-issue` | 项目级 |
| `.claude/commands/commit-push-pr.md` | `/commit-push-pr` | 项目级 |
| `.claude/commands/dedupe.md` | `/dedupe` | 项目级 |

---

## 2. Command Frontmatter 规范

每个命令文件的 YAML frontmatter 控制命令的元信息和执行权限。

### 2.1 所有 Frontmatter 字段对比表

| 命令 | `description` | `argument-hint` | `allowed-tools` | `hide-from-slash-command-tool` |
|------|--------------|----------------|-----------------|-------------------------------|
| `feature-dev` | ✅ 有 | `Optional feature description` | ❌ 无限制 | ❌ |
| `create-plugin` | ✅ 有 | `Optional plugin description` | ✅ 明确列出 7 个 | ❌ |
| `hookify` | ✅ 有 | `Optional specific behavior to address` | ✅ 明确列出 7 个 | ❌ |
| `hookify:configure` | ✅ 有 | ❌ 无 | ✅ 明确列出 5 个 | ❌ |
| `hookify:list` | ✅ 有 | ❌ 无 | ✅ 明确列出 3 个 | ❌ |
| `hookify:help` | ✅ 有 | ❌ 无 | ✅ 仅 `["Read"]` | ❌ |
| `commit` | ✅ 有 | ❌ 无 | ✅ 精确 Bash 子命令 | ❌ |
| `commit-push-pr` (plugin) | ✅ 有 | ❌ 无 | ✅ 精确 Bash 子命令 | ❌ |
| `clean_gone` | ✅ 有 | ❌ 无 | ❌ 无限制 | ❌ |
| `review-pr` | ✅ 有 | `[review-aspects]` | ✅ 明确列出 5 个 | ❌ |
| `code-review` | ✅ 有 | ❌ 无 | ✅ 精确 Bash+MCP | ❌ |
| `ralph-loop` | ✅ 有 | `PROMPT [--max-iterations N] [--completion-promise TEXT]` | ✅ 仅 setup 脚本 | `"true"` |
| `cancel-ralph` | ✅ 有 | ❌ 无 | ✅ 精确 Bash 文件操作 | `"true"` |
| `ralph-wiggum:help` | ✅ 有 | ❌ 无 | ❌ 无限制 | ❌ |
| `new-sdk-app` | ✅ 有 | `[project-name]` | ❌ 无限制 | ❌ |
| `triage-issue` | ✅ 有 | ❌ 无 | ✅ 仅特定脚本 | ❌ |
| `commit-push-pr` (项目) | ✅ 有 | ❌ 无 | ✅ 精确 Bash 子命令 | ❌ |
| `dedupe` | ✅ 有 | ❌ 无 | ✅ 仅特定脚本 | ❌ |

### 2.2 各字段详细说明

#### `description`（必填）
所有 18 个命令均有 `description` 字段，用于在 `/help` 列表中展示简要说明。

```yaml
---
description: Guided feature development with codebase understanding and architecture focus
---
```

#### `argument-hint`（可选）
提供给用户的参数提示，显示在命令补全界面中。带参命令示例：

```yaml
---
argument-hint: "PROMPT [--max-iterations N] [--completion-promise TEXT]"
---
```

`$ARGUMENTS` 占位符在命令正文中被替换为用户实际输入：

```markdown
Initial request: $ARGUMENTS
```

条件判断 `$ARGUMENTS` 是否存在：

```markdown
**If $ARGUMENTS is provided:**
- User has given specific instructions: `$ARGUMENTS`

**If $ARGUMENTS is empty:**
- Launch the conversation-analyzer agent to find problematic behaviors
```

#### `allowed-tools`（权限白名单）
两种形式：
1. **工具名列表**（通配）：`["Read", "Write", "Bash", "Grep", "Glob", "TodoWrite", "Task"]`
2. **精确 Bash 子命令**（最小权限原则）：

```yaml
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
```

`triage-issue` 和 `dedupe` 仅允许调用特定的封装脚本，彻底屏蔽通用 Bash：

```yaml
allowed-tools: Bash(./scripts/gh.sh:*), Bash(./scripts/edit-issue-labels.sh:*)
```

#### `hide-from-slash-command-tool`（隐藏标志）
`ralph-loop` 和 `cancel-ralph` 使用 `hide-from-slash-command-tool: "true"` 隐藏命令，防止 Claude 在 Task 工具内部自动触发这些命令：

```yaml
hide-from-slash-command-tool: "true"
```

#### 命令内的 Shell 预执行（`!` 语法）
`commit` 和 `commit-push-pr` 命令利用 `!` 前缀在命令加载时立即执行 Shell 命令，将结果注入上下文：

```markdown
## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged changes): !`git diff HEAD`
- Current branch: !`git branch --show-current`
```

---

## 3. 多阶段工作流命令设计

### 3.1 feature-dev 7 阶段工作流

#### 完整流程图

```
用户输入: /feature-dev [可选描述]
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: Discovery（发现阶段）                              │
│  - 创建 TodoWrite 任务清单（全部7阶段）                       │
│  - 如需求不清，向用户提问3个关键问题                          │
│  - 确认理解并等待用户确认                                     │
└──────────────────────┬──────────────────────────────────────┘
                       │ 用户确认理解 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 2: Codebase Exploration（代码库探索阶段）             │
│  - 并行启动 2-3 个 code-explorer agent                       │
│    • Agent A: 探索相似特性实现                               │
│    • Agent B: 架构和抽象层分析                               │
│    • Agent C: 现有功能/UI模式分析                            │
│  - 读取 Agent 返回的重要文件（5-10个）                        │
│  - 输出综合探索报告                                           │
└──────────────────────┬──────────────────────────────────────┘
                       │ 自动继续 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 3: Clarifying Questions（澄清问题阶段）  ⚠️ 关键     │
│  - 整理所有未明确点：边界情况、错误处理、集成点、兼容性       │
│  - 向用户展示问题清单                                         │
│  ⏸️ 等待用户回答后继续                                       │
│  - 如用户说"你来定"，需给出建议并请求明确确认                  │
└──────────────────────┬──────────────────────────────────────┘
                       │ 用户回答问题 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 4: Architecture Design（架构设计阶段）                │
│  - 并行启动 2-3 个 code-architect agent                      │
│    • 方案A: 最小改动（最大化复用）                            │
│    • 方案B: 清洁架构（可维护性优先）                          │
│    • 方案C: 实用平衡（速度+质量）                             │
│  - 整合各方案，给出推荐并说明理由                             │
│  ⏸️ 等待用户选择方案                                         │
└──────────────────────┬──────────────────────────────────────┘
                       │ 用户选择方案 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 5: Implementation（实现阶段）   ⛔ 需明确授权         │
│  - DO NOT START WITHOUT USER APPROVAL                        │
│  - 读取所有前序阶段识别的相关文件                             │
│  - 按选定架构实现                                             │
│  - 严格遵循代码库惯例                                         │
│  - 更新 TodoWrite 进度                                        │
└──────────────────────┬──────────────────────────────────────┘
                       │ 实现完成 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 6: Quality Review（质量评审阶段）                     │
│  - 并行启动 3 个 code-reviewer agent                         │
│    • 评审器1: 简洁性/DRY/优雅性                               │
│    • 评审器2: Bug/功能正确性                                  │
│    • 评审器3: 项目规范/抽象层                                 │
│  - 整合问题，标注严重程度                                      │
│  ⏸️ 展示问题，询问用户处理意见（立即修复/后续修复/忽略）       │
└──────────────────────┬──────────────────────────────────────┘
                       │ 问题处理完毕 ↓
┌──────────────────────▼──────────────────────────────────────┐
│  Phase 7: Summary（总结阶段）                                │
│  - 标记所有 Todo 完成                                         │
│  - 汇总：构建内容、关键决策、修改文件、后续建议               │
└─────────────────────────────────────────────────────────────┘
```

#### 关键设计约定

1. **人工审批门（Human Approval Gates）**：阶段 1 后、阶段 3 后、阶段 4 后、阶段 5 前均强制等待用户确认
2. **并行 Agent 策略**：探索、设计、评审三个阶段均使用并行 Agent，提升效率
3. **Agent 返回文件清单**：Agent 完成后，主流程必须读取 Agent 识别的关键文件

```markdown
**Read files identified by agents**: When launching agents, ask them to return lists of
the most important files to read. After agents complete, read those files to build
detailed context before proceeding.
```

---

### 3.2 create-plugin 8 阶段引导工作流

`plugin-dev:create-plugin` 在 `feature-dev` 的基础上增加了两个额外阶段：

```
Phase 1: Discovery（需求发现）
    │ ⏸️ 用户确认
Phase 2: Component Planning（组件规划）← 必须加载 plugin-structure 技能
    │ ⏸️ 用户确认组件清单
Phase 3: Detailed Design & Clarifying Questions（详细设计+澄清）
    │ ⏸️ 用户确认规格
Phase 4: Plugin Structure Creation（目录结构创建）
    │ 自动继续
Phase 5: Component Implementation（组件实现）← 按需加载对应技能
    │ 自动继续
Phase 6: Validation & Quality Check（验证+质量检查）
    │ ⏸️ 用户决定是否修复问题
Phase 7: Testing & Verification（测试验证）
    │ ⏸️ 用户决定是否引导测试
Phase 8: Documentation & Next Steps（文档+后续步骤）
```

**独特设计点**：
- `allowed-tools` 明确包含 `AskUserQuestion` 和 `Skill`，确保工具调用权限
- 阶段 5 按照不同组件类型（Skills/Commands/Agents/Hooks/MCP）动态加载对应技能
- 使用专业 Agent（`agent-creator`、`plugin-validator`、`skill-reviewer`）辅助开发

组件规划输出为结构化表格：

```markdown
| Component Type | Count | Purpose |
|----------------|-------|---------|
| Skills         | 2     | Hook patterns, MCP usage |
| Commands       | 3     | Deploy, configure, validate |
| Agents         | 1     | Autonomous validation |
```

---

### 3.3 new-sdk-app 逐问式引导工作流

`agent-sdk-dev:new-sdk-app` 采用完全不同的"一次只问一个问题"模式：

```
Step 1: 问语言（TypeScript/Python） → 等待回答
Step 2: 问项目名 → 等待回答（$ARGUMENTS 可跳过此步）
Step 3: 问 Agent 类型 → 等待回答
Step 4: 问起始点（Hello World/基础/自定义）→ 等待回答
Step 5: 确认工具链（npm/pnpm/bun）→ 等待回答
↓ 所有问题回答完毕后
创建项目结构 → 安装依赖 → 类型检查 → 验证
```

明确规定 **"IMPORTANT: Ask these questions one at a time"**，这是与 feature-dev 批量提问不同的设计选择。

---

## 4. 交互式命令设计模式

### 4.1 "等待用户回答再继续"的设计约定

Claude Code 命令系统中存在几种等待用户输入的设计方式：

#### 方式 A：文本约定（显式等待）
直接在 Markdown 中写明等待：

```markdown
**Wait for answers before proceeding to architecture design**
```

```markdown
**DO NOT START WITHOUT USER APPROVAL**
```

`feature-dev` 中阶段 3 标注为 **CRITICAL: This is one of the most important phases. DO NOT SKIP.**

#### 方式 B：AskUserQuestion 工具（结构化交互）
`hookify` 命令在 `allowed-tools` 中声明 `AskUserQuestion`，并在命令体内描述其调用方式：

```json
{
  "questions": [
    {
      "question": "Which behaviors to hookify?",
      "header": "Create Rules",
      "multiSelect": true,
      "options": [
        {
          "label": "Block rm -rf",
          "description": "Prevents dangerous file deletion"
        }
      ]
    }
  ]
}
```

支持 `multiSelect: true` 实现多选交互。

#### 方式 C：单问式等待（new-sdk-app）
明确约定每次只问一个问题，每次都等待：

```markdown
IMPORTANT: Ask these questions one at a time. Wait for the user's response
before asking the next question.
```

---

### 4.2 hookify 命令集合模式（命名空间命令）

hookify 插件展示了最完整的"命名空间命令集合"设计：

```
/hookify              ← 主命令（创建规则）
/hookify:help         ← 帮助子命令
/hookify:list         ← 列表子命令
/hookify:configure    ← 配置子命令
```

每个子命令职责单一，工具权限最小化：

```
hookify.md:     7 个工具（完整权限，包含 Task 启动 Agent）
configure.md:   5 个工具（Read/Edit/Glob/AskUserQuestion/Skill）
list.md:        3 个工具（Glob/Read/Skill）
help.md:        1 个工具（仅 Read）
```

#### 规则文件命名约定

hookify 创建的规则文件采用统一格式：`.claude/hookify.{rule-name}.local.md`

Glob 模式：`.claude/hookify.*.local.md`

规则文件 frontmatter：

```yaml
---
name: warn-dangerous-rm
enabled: true
event: bash        # bash | file | stop | prompt | all
pattern: rm\s+-rf
action: warn       # warn | block
---
```

命令体（Message Body）即触发后展示给 Claude 的提示内容。

---

### 4.3 ralph-wiggum 命令集合（loop 控制模式）

ralph-wiggum 实现了不同的命令集合模式——控制长期运行循环的 **Start/Cancel/Help** 三元组：

```
/ralph-wiggum:ralph-loop  ← 启动循环（配置并开始）
/ralph-wiggum:cancel-ralph ← 取消循环（状态文件清理）
/ralph-wiggum:help         ← 使用说明
```

`ralph-loop` 的独特 frontmatter：

```yaml
allowed-tools: ["Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh:*)"]
hide-from-slash-command-tool: "true"
```

- 仅允许调用单个脚本
- 隐藏命令防止自动触发
- 使用 `${CLAUDE_PLUGIN_ROOT}` 变量确保路径可移植

脚本调用使用 `!` 语法立即执行：

```markdown
```!
"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh" $ARGUMENTS
```
```

`cancel-ralph` 精确控制 Bash 权限：

```yaml
allowed-tools: ["Bash(test -f .claude/ralph-loop.local.md:*)", "Bash(rm .claude/ralph-loop.local.md)", "Read(.claude/ralph-loop.local.md)"]
```

---

## 5. 项目级命令与全局命令

### 5.1 .claude/commands/ 项目级命令

项目级命令存放在仓库根目录的 `.claude/commands/`，仅在该项目中可用。Claude Code 仓库自身的 3 个项目命令展示了实际工程场景中的用法。

#### triage-issue.md — CI 自动化 Issue 分类

这是最接近"自动化脚本"的命令设计：

```yaml
---
allowed-tools: Bash(./scripts/gh.sh:*),Bash(./scripts/edit-issue-labels.sh:*)
description: Triage GitHub issues by analyzing and applying labels
---
```

**设计特点**：
- 完全限制在两个封装脚本，屏蔽通用 Bash 和网络访问
- 通过 `$ARGUMENTS` 接收来自 GitHub Actions 注入的上下文（Event 类型、Issue 编号等）
- 区分两种事件类型：`issues`（新建）和 `issue_comment`（评论）

工作流程：

```
读取标签列表 → 读取 Issue → 读取评论
    ↓
新建 Issue？→ 检查合法性 → 分析分类 → 应用标签
评论事件？ → 评估生命周期标签（移除过期/添加缺失）
```

#### commit-push-pr.md — 一键提交发布

```yaml
---
allowed-tools: Bash(git checkout --branch:*), Bash(git add:*), Bash(git status:*),
               Bash(git push:*), Bash(git commit:*), Bash(gh pr create:*)
description: Commit, push, and open a PR
---
```

**设计特点**：
- `!` 语法预注入上下文（`git status`、`git diff HEAD`、`git branch --show-current`）
- 要求 Claude 在**单次响应**中完成所有工具调用（原子性）：

```markdown
You MUST do all of the above in a single message. Do not use any other tools
or do anything else.
```

#### dedupe.md — 并行 Agent Issue 去重

```markdown
1. 检查 Issue 是否需要去重（单 Agent）
2. 读取 Issue 内容并生成摘要（单 Agent）
3. 并行启动 5 个 Agent 搜索重复 Issue（使用多样化关键词）
4. 整合结果过滤误报（单 Agent）
5. 执行评论脚本
```

这是项目命令中最复杂的 Agent 编排：5 个搜索 Agent 并行 + 结果过滤 Agent 串行。

---

### 5.2 项目级 vs 插件级命令的设计差异

| 维度 | 项目级（.claude/commands/） | 插件级（plugins/*/commands/） |
|------|---------------------------|------------------------------|
| 工具限制 | 非常严格（仅封装脚本） | 相对宽松（列举工具类型） |
| 使用场景 | CI/自动化、DevOps 工作流 | 开发辅助、引导式工作流 |
| 阶段数量 | 1-5 步（简洁直接） | 4-8 个阶段（系统完整） |
| 用户交互 | 极少（自动化为主） | 频繁（等待确认） |
| `argument-hint` | 通常不设 | 常见 |
| 预执行 Shell（`!`） | 常见 | 较少 |

---

## 6. 并行 Agent 启动模式

Claude Code 命令中有两种方式描述并行 Agent 启动：

### 6.1 文本描述式（feature-dev 风格）

```markdown
**Actions**:
1. Launch 2-3 code-explorer agents in parallel. Each agent should:
   - Trace through the code comprehensively
   - Target a different aspect of the codebase
   - Include a list of 5-10 key files to read
```

通过自然语言约定 Agent 数量和职责分工。

### 6.2 Task 工具调用式（hookify 风格）

```markdown
Use the Task tool to launch conversation-analyzer agent:
```json
{
  "subagent_type": "general-purpose",
  "description": "Analyze conversation for unwanted behaviors",
  "prompt": "..."
}
```
```

提供完整的 Task 工具参数，精确控制 Agent 类型、描述和 prompt。

### 6.3 code-review 的分层 Agent 管道

`code-review:code-review` 命令展示了最复杂的 Agent 管道设计：

```
Level 1（串行）: haiku Agent — 检查 PR 是否需要审查
Level 1（串行）: haiku Agent — 收集 CLAUDE.md 文件路径
Level 1（串行）: sonnet Agent — 生成 PR 变更摘要
Level 2（并行）: 4 个 Agent
    ├─ sonnet Agent × 2 — CLAUDE.md 合规检查（并行）
    ├─ opus Agent — Bug 扫描
    └─ opus Agent — 安全/逻辑问题扫描
Level 3（并行）: N 个 Agent — 各问题的独立验证
Level 4（串行）: 过滤误报 → 输出摘要 → 发布内联评论
```

根据问题类型使用不同模型：
- **haiku**：轻量预检（快速低成本）
- **sonnet**：规范合规检查（平衡）
- **opus**：深度 Bug/安全分析（高质量）

---

## 7. 关键发现总结

### 发现 1：命令文件即完整的 LLM Prompt

命令文件的正文（frontmatter 以下部分）直接作为系统指令传递给 Claude。没有中间解析层——Markdown 的结构性（标题、列表、代码块）就是给 LLM 的格式化提示。这意味着：
- 命令的质量等于 prompt engineering 的质量
- Markdown 标题（`## Phase X`）充当思维框架
- 代码块用于展示期望的工具调用格式

### 发现 2：`allowed-tools` 是核心安全机制

系统通过 `allowed-tools` 实现最小权限原则。特别是 `Bash(specific-script:*)` 语法将 Bash 执行限制在特定脚本，防止命令被滥用执行任意 Shell 命令。CI 场景的命令（`triage-issue`、`dedupe`）完全封闭在封装脚本内。

### 发现 3：三种并行 Agent 模式

| 模式 | 典型命令 | 特点 |
|------|---------|------|
| 探索并行（Exploration） | `feature-dev` Phase 2 | 多角度覆盖，每个 Agent 聚焦不同方面 |
| 设计并行（Design） | `feature-dev` Phase 4 | 多方案竞争，权衡利弊后由用户选择 |
| 评审并行（Review） | `feature-dev` Phase 6 | 分工专业化，各自聚焦特定质量维度 |

### 发现 4：命令命名空间形成功能组

hookify 和 ralph-wiggum 展示了命名空间命令集合模式：一个主命令 + 多个功能子命令（`help`/`list`/`configure`）形成完整的命令族。子命令的工具权限随功能减少而收窄，遵循最小权限原则。

### 发现 5：`$ARGUMENTS` 的双重用途

- **用户输入转发**：将命令参数直接传入 prompt（如 `feature-dev` 的特性描述、`hookify` 的行为描述）
- **条件分支控制**：判断 `$ARGUMENTS` 是否为空来决定工作流路径（有参数走快速路径，无参数走探索路径）

### 发现 6：状态管理通过 .local.md 文件实现

hookify 规则（`.claude/hookify.*.local.md`）和 ralph-loop 状态（`.claude/ralph-loop.local.md`）均使用项目目录中的本地文件存储运行时状态。这是一种无服务器的轻量状态管理方案，与 Claude Code 的文件系统工具无缝集成。

### 发现 7：`!` 语法实现命令加载时上下文注入

`commit`、`commit-push-pr` 类命令通过 `!` 前缀在命令加载时立即执行 Shell 命令，将 `git status`、`git diff HEAD` 等上下文注入到命令 prompt 中，使 Claude 在开始任何推理之前就拥有完整的环境上下文。

### 发现 8：多阶段命令中的强制等待点设计

feature-dev 在关键节点设置了明确的强制等待机制：
- **CRITICAL** 标注阶段不可跳过（Phase 3）
- **DO NOT START WITHOUT USER APPROVAL**（Phase 5）
- 用户输入模糊时要求明确确认，不允许假设（"whatever you think is best" → 给出推荐并请求确认）

这种设计确保了 LLM 不会在关键决策点擅自继续，防止昂贵的实现工作建立在错误假设之上。
