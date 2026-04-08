# Claude Code 核心工作流设计模式分析

> **文档范围**：深度分析 claude-code 仓库中五大核心工作流插件的设计模式，包括 feature-dev、code-review、plugin-dev、agent-sdk-dev、ralph-wiggum。

---

## 目录

1. [Feature Development 七阶段工作流](#1-feature-development-七阶段工作流)
2. [Code Review 并行 Agent 模式](#2-code-review-并行-agent-模式)
3. [Agent SDK 开发工作流](#3-agent-sdk-开发工作流)
4. [Ralph Wiggum 自我迭代循环](#4-ralph-wiggum-自我迭代循环)
5. [Plugin 开发八阶段工作流](#5-plugin-开发八阶段工作流)
6. [工作流模式横向对比](#6-工作流模式横向对比)
7. [关键发现总结](#7-关键发现总结)

---

## 1. Feature Development 七阶段工作流

**来源文件**：
- `plugins/feature-dev/commands/feature-dev.md`
- `plugins/feature-dev/agents/code-explorer.md`
- `plugins/feature-dev/agents/code-architect.md`
- `plugins/feature-dev/agents/code-reviewer.md`

### 1.1 七阶段流程全貌

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Feature Development 七阶段流程                    │
└─────────────────────────────────────────────────────────────────────┘

Phase 1: Discovery（发现）
  ┌───────────────────────────────┐
  │  理解需求 → 确认理解 → 创建 Todo │
  └───────────────────────────────┘
              │
              ▼
Phase 2: Codebase Exploration（代码库探索）
  ┌───────────────────────────────────────────────────┐
  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
  │  │ code-explorer │  │ code-explorer │  │code-    │ │
  │  │ (相似功能)    │  │ (架构模式)    │  │explorer │ │
  │  │  [并行]       │  │  [并行]       │  │(UX模式) │ │
  │  └──────┬───────┘  └──────┬───────┘  └────┬────┘ │
  │         └─────────────────┼───────────────┘       │
  │                           ▼                       │
  │              读取 Agent 返回的关键文件列表           │
  └───────────────────────────────────────────────────┘
              │
              ▼
Phase 3: Clarifying Questions（澄清问题）★ CRITICAL
  ┌───────────────────────────────────────────────────┐
  │  识别未明确点 → 向用户提问 → 等待用户回答           │
  │  （边界案例/错误处理/集成点/性能/向后兼容）         │
  └───────────────────────────────────────────────────┘
              │
              ▼ 用户确认后继续
Phase 4: Architecture Design（架构设计）
  ┌───────────────────────────────────────────────────┐
  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
  │  │ code-architect│  │ code-architect│  │code-    │ │
  │  │ (最小变更)    │  │ (干净架构)    │  │architect│ │
  │  │  [并行]       │  │  [并行]       │  │(实用平衡)│ │
  │  └──────┬───────┘  └──────┬───────┘  └────┬────┘ │
  │         └─────────────────┼───────────────┘       │
  │                           ▼                       │
  │      对比方案 → 给出推荐 → 等待用户选择             │
  └───────────────────────────────────────────────────┘
              │
              ▼ 用户明确批准后继续
Phase 5: Implementation（实现）★ 需要用户明确批准
  ┌───────────────────────────────────────────────────┐
  │  读取相关文件 → 遵循架构实现 → 更新 Todo            │
  └───────────────────────────────────────────────────┘
              │
              ▼
Phase 6: Quality Review（质量审查）
  ┌───────────────────────────────────────────────────┐
  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
  │  │ code-reviewer │  │ code-reviewer │  │code-    │ │
  │  │ (简洁/DRY)    │  │ (Bug/功能正确)│  │reviewer │ │
  │  │  [并行]       │  │  [并行]       │  │(项目惯例)│ │
  │  └──────┬───────┘  └──────┬───────┘  └────┬────┘ │
  │         └─────────────────┼───────────────┘       │
  │                           ▼                       │
  │   汇总发现 → 向用户展示 → 询问如何处理              │
  └───────────────────────────────────────────────────┘
              │
              ▼
Phase 7: Summary（总结）
  ┌───────────────────────────────────────────────────┐
  │  标记 Todo 完成 → 总结建设内容 → 记录关键决策       │
  └───────────────────────────────────────────────────┘
```

### 1.2 核心原则：「先问后做」

feature-dev 的核心设计哲学是"在行动之前深度理解"。命令文件明确标注：

```
## Phase 3: Clarifying Questions
CRITICAL: This is one of the most important phases. DO NOT SKIP.

## Phase 5: Implementation
DO NOT START WITHOUT USER APPROVAL
```

这种「先问后做」原则在三个维度上体现：

1. **阶段 1→2 转换**：先让 Agent 探索代码，再基于探索结果确认理解
2. **阶段 2→3 转换**：先探索再提问，而非假设——避免在错误基础上设计
3. **阶段 4→5 转换**：架构设计必须获得用户显式批准才能开始实现

### 1.3 Agent 组合策略

| 阶段 | Agent | 数量 | 并行策略 | 关注点 |
|------|-------|------|----------|--------|
| Phase 2 | code-explorer | 2-3 | 完全并行 | 不同维度（相似功能/架构/UX）|
| Phase 4 | code-architect | 2-3 | 完全并行 | 不同方案（最小/干净/平衡）|
| Phase 6 | code-reviewer | 3 | 完全并行 | 不同视角（简洁/Bug/惯例）|

**Agent 间的信息传递**：Phase 2 的 code-explorer 需返回"5-10 个关键文件列表"，主 Agent 在继续前须亲自读取这些文件，避免信息在 Agent 间传递时失真。

### 1.4 code-explorer Agent 设计

```yaml
name: code-explorer
model: sonnet
color: yellow
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
```

**四层分析框架**：
1. **Feature Discovery**：找入口点（API/UI/CLI）
2. **Code Flow Tracing**：追踪调用链到输出
3. **Architecture Analysis**：映射抽象层级
4. **Implementation Details**：关键算法与边界情况

输出要求包含必须有"**绝对关键文件列表**"——这是工作流中人机协作的关键接口点。

### 1.5 code-architect Agent 设计

```yaml
name: code-architect
model: sonnet
color: green
```

与 code-explorer 不同，code-architect 被要求"做出决定性选择——选一个方案并坚持"，而不是"提出多种选择"。这体现了角色分工：探索时用多并行，架构时用单决策，呈现给用户对比。

### 1.6 code-reviewer 置信度过滤机制

```yaml
name: code-reviewer
model: sonnet
color: red
```

采用 0-100 置信度评分，**只汇报置信度 ≥ 80 的问题**：

| 置信度 | 含义 |
|--------|------|
| 0 | 误报，不应标记 |
| 25 | 可能是问题，可能是误报 |
| 50 | 真实问题，但是次要 |
| 75 | 高置信度，真实且重要 |
| 100 | 绝对确定，频繁发生 |

这种设计解决了 AI 代码审查"噪音过多"的核心问题。

---

## 2. Code Review 并行 Agent 模式

**来源文件**：
- `plugins/code-review/commands/code-review.md`
- `plugins/pr-review-toolkit/commands/review-pr.md`
- `plugins/pr-review-toolkit/agents/`（全部 6 个 Agent）

### 2.1 两套 Code Review 系统对比

| 特性 | code-review 插件 | pr-review-toolkit 插件 |
|------|-----------------|----------------------|
| 使用场景 | GitHub PR 自动审查 | 本地代码变更审查 |
| Agent 数量 | 动态（4-6 个） | 6 个专项 Agent |
| 并行方式 | 4 个 Agent 完全并行 | 可顺序或并行 |
| 置信度过滤 | 有（Haiku 前置过滤 + 并行验证）| 有（每个 Agent 内置） |
| 输出目标 | GitHub PR 评论 | 终端报告 |
| 验证步骤 | 明确二次验证层 | 各 Agent 自验证 |

### 2.2 code-review 插件的多层 Agent 架构

```
┌───────────────────────────────────────────────────────────┐
│                   code-review 工作流                       │
└───────────────────────────────────────────────────────────┘

Step 1: Haiku Agent（前置过滤）
  ├── PR 是否已关闭/草稿？
  ├── Claude 是否已评论过？
  └── 是否需要审查？ → 不需要则停止

Step 2: Haiku Agent（文件发现）
  └── 返回相关 CLAUDE.md 文件路径列表

Step 3: Sonnet Agent（PR 摘要）
  └── 返回变更摘要

Step 4: 4 个 Agent 完全并行 ──────────────────────────────┐
  ├── Sonnet Agent 1: CLAUDE.md 合规审查                   │
  ├── Sonnet Agent 2: CLAUDE.md 合规审查（并行）            │
  ├── Opus Agent 3:  Bug 扫描（仅 diff 范围）              │
  └── Opus Agent 4:  安全/逻辑问题（仅变更代码）           │
                                                           │
Step 5: 针对每个发现问题，并行验证 ◄────────────────────────┘
  ├── 验证 Bug（Opus Agent）
  └── 验证 CLAUDE.md 违规（Sonnet Agent）

Step 6: 过滤未通过验证的问题 → 高信噪比结果

Step 7: 输出到终端（无 --comment 参数）
  或 → Post GitHub inline 评论（有 --comment 参数）
```

**关键设计洞察**：二次验证层（Step 5）是这个工作流的核心创新。每个问题被独立 Agent 验证，并非简单的置信度阈值过滤，而是真正的"同行评审"机制。

### 2.3 pr-review-toolkit 的 6 个专项 Agent

```
┌────────────────────────────────────────────────────────────┐
│                  pr-review-toolkit Agent 分工               │
└────────────────────────────────────────────────────────────┘

┌──────────────────┐  职责：Bug + CLAUDE.md 合规
│  code-reviewer   │  模型：Opus
│  (绿色)          │  特点：置信度 ≥80 才汇报
└──────────────────┘

┌──────────────────┐  职责：代码简化和可读性
│  code-simplifier │  模型：Opus
│  (绿色)          │  特点：保持功能不变，只改写法
└──────────────────┘

┌──────────────────┐  职责：注释准确性和维护性
│ comment-analyzer │  模型：inherit
│  (绿色)          │  特点：防止注释腐化
└──────────────────┘

┌──────────────────┐  职责：测试覆盖质量
│ pr-test-analyzer │  模型：inherit
│  (青色)          │  特点：行为覆盖 > 行覆盖
└──────────────────┘

┌──────────────────┐  职责：静默失败检测
│silent-failure-   │  模型：inherit
│hunter (黄色)     │  特点：零容忍空 catch 块
└──────────────────┘

┌──────────────────┐  职责：类型设计质量
│type-design-      │  模型：inherit
│analyzer (粉色)   │  特点：四维度评分（封装/不变量）
└──────────────────┘
```

### 2.4 pr-review-toolkit 的调度策略

`review-pr` 命令实现了智能的按需调度：

```
根据变更内容决定激活哪些 Agent：
├── 始终激活：code-reviewer
├── 测试文件变更 → pr-test-analyzer
├── 注释/文档变更 → comment-analyzer
├── 错误处理变更 → silent-failure-hunter
├── 新增类型 → type-design-analyzer
└── 审查通过后 → code-simplifier（可选收尾）
```

**两种执行模式**：
- **顺序模式**（默认）：逐个 Agent 运行，便于交互理解
- **并行模式**（用户请求）：同时启动所有 Agent，速度优先

### 2.5 silent-failure-hunter 的审查维度

这个 Agent 体现了专项 Agent 的深度价值。它有五个**不可妥协的原则**：

1. 静默失败不可接受（必须有日志 + 用户反馈）
2. 用户应获得可操作的错误反馈
3. 降级行为必须明确且有理由
4. Catch 块必须有明确类型（不允许宽泛捕获）
5. Mock 实现只属于测试代码

### 2.6 type-design-analyzer 的四维评分体系

对每个类型从四个维度评分（1-10）：

| 维度 | 评估内容 |
|------|---------|
| **封装性（Encapsulation）** | 内部实现是否隐藏，外部是否能破坏不变量 |
| **不变量表达（Invariant Expression）** | 通过类型结构表达约束的清晰度 |
| **不变量有用性（Invariant Usefulness）** | 不变量是否防止真实 Bug |
| **不变量强制（Invariant Enforcement）** | 构造时是否检查，变更点是否守护 |

---

## 3. Agent SDK 开发工作流

**来源文件**：
- `plugins/agent-sdk-dev/commands/new-sdk-app.md`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md`

### 3.1 交互式一问一答初始化流程

`/new-sdk-app` 命令实现了严格的单问题问答模式：

```
┌────────────────────────────────────────────────────────┐
│              /new-sdk-app 交互式初始化流程               │
└────────────────────────────────────────────────────────┘

IMPORTANT: Ask these questions one at a time.
Wait for the user's response before asking the next question.

问题 1: 语言选择
  "Would you like to use TypeScript or Python?"
  └── 等待回答 ──────────────────────────────────────────┐
                                                         │
问题 2: 项目名称（若 $ARGUMENTS 已提供则跳过）            │◄──
  "What would you like to name your project?"
  └── 等待回答 ──────────────────────────────────────────┐
                                                         │
问题 3: Agent 类型（若问题 2 已包含足够细节则跳过）        │◄──
  "What kind of agent are you building?"
  (coding/business/custom)
  └── 等待回答 ──────────────────────────────────────────┐
                                                         │
问题 4: 起步模板                                         │◄──
  "Would you like: Hello World / basic / specific?"
  └── 等待回答 ──────────────────────────────────────────┐
                                                         │
问题 5: 工具链确认（npm/yarn/pnpm/bun）                  │◄──
  确认工具链选择
  └── 等待回答 ──────────────────────────────────────────┐
                                                         │
  ──────────────────────────────────────────────────────►│
创建 Setup Plan → 获取最新包版本 → 安装 SDK → 创建文件
              │
              ▼
VERIFY THE CODE WORKS BEFORE FINISHING:
  ├── TypeScript：运行 `npx tsc --noEmit`，修复所有类型错误
  └── Python：验证语法和导入
              │
              ▼
启动 verifier Agent（按语言选择）
  ├── TypeScript → agent-sdk-verifier-ts
  └── Python     → agent-sdk-verifier-py
```

### 3.2 文档先行原则

在提问之前，命令要求先读取官方文档：

```markdown
Before starting, review the official documentation:
1. Start with the overview: https://docs.claude.com/en/api/agent-sdk/overview
2. Based on the user's language choice, read the appropriate SDK reference
3. Read relevant guides based on the user's needs
```

这确保了 Agent 基于最新 API 文档而非训练数据来引导用户，避免过时信息。

### 3.3 verifier Agent 的验证机制

两个 verifier Agent（TS/Python）采用相同的验证框架，但针对各语言特性定制：

**TypeScript verifier 的 9 个验证维度**：

| 维度 | 验证内容 |
|------|---------|
| SDK 安装配置 | `@anthropic-ai/claude-agent-sdk` 版本，`"type": "module"` |
| TypeScript 配置 | tsconfig.json 的模块解析，ES 模块支持 |
| SDK 使用模式 | 正确 import，Agent 初始化，流/单次模式 |
| 类型安全 | 运行 `npx tsc --noEmit`，零错误才通过 |
| 脚本配置 | package.json scripts（build/start/typecheck）|
| 环境安全 | `.env.example` 存在，`.env` 在 `.gitignore` |
| SDK 最佳实践 | 系统提示质量，权限范围，MCP 集成 |
| 功能验证 | Agent 初始化和执行流程正确 |
| 文档 | README，设置说明 |

**报告格式**：PASS / PASS WITH WARNINGS / FAIL（三态结果，清晰可操作）

---

## 4. Ralph Wiggum 自我迭代循环

**来源文件**：
- `plugins/ralph-wiggum/commands/ralph-loop.md`
- `plugins/ralph-wiggum/commands/cancel-ralph.md`
- `plugins/ralph-wiggum/hooks/hooks.json`
- `plugins/ralph-wiggum/hooks/stop-hook.sh`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`

### 4.1 自迭代循环的核心机制

```
┌──────────────────────────────────────────────────────────────┐
│                 Ralph Wiggum 自迭代机制                       │
└──────────────────────────────────────────────────────────────┘

用户运行 /ralph-loop PROMPT --max-iterations N
                     │
                     ▼
setup-ralph-loop.sh 执行
  ├── 解析参数（--max-iterations, --completion-promise）
  ├── 创建状态文件 .claude/ralph-loop.local.md
  │   ┌──────────────────────────────────┐
  │   │ ---                              │
  │   │ active: true                    │
  │   │ iteration: 1                    │
  │   │ max_iterations: N               │
  │   │ completion_promise: "TEXT"       │
  │   │ started_at: "2026-04-08T..."    │
  │   │ ---                             │
  │   │                                 │
  │   │ [原始 PROMPT 文本]               │
  │   └──────────────────────────────────┘
  └── 输出初始 PROMPT，Claude 开始工作
                     │
                     ▼
  Claude 完成工作，尝试退出会话
                     │
                     ▼ Stop 事件触发
hooks.json 拦截：Stop → stop-hook.sh
  │
  ├── 检查 .claude/ralph-loop.local.md 是否存在
  │   └── 不存在 → exit 0（允许退出）
  │
  ├── 读取状态：iteration, max_iterations, completion_promise
  │
  ├── 检查迭代次数：iteration >= max_iterations → 清除状态，允许退出
  │
  ├── 从 transcript 读取最后一条 assistant 消息
  │   └── 检查是否包含 <promise>COMPLETION_PROMISE</promise>
  │       └── 匹配 → 清除状态，允许退出 ✅
  │
  ├── 未完成 → 更新 iteration: N+1
  │
  └── 输出 JSON 阻断退出：
      {
        "decision": "block",
        "reason": "[原始 PROMPT 文本]",  ← 喂回给 Claude
        "systemMessage": "🔄 Ralph iteration N+1 | ..."
      }
```

### 4.2 Stop Hook 的技术实现

`hooks.json` 配置极简：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh"
          }
        ]
      }
    ]
  }
}
```

`stop-hook.sh` 通过输出 `decision: "block"` 的 JSON 来阻断退出，并将原始 Prompt 作为 `reason` 字段返回给 Claude，形成闭环。

### 4.3 完成承诺（Completion Promise）机制

这是防止 Claude"说谎退出"的核心设计：

```bash
# 从最后一条 assistant 消息中提取 <promise> 标签内容
PROMISE_TEXT=$(echo "$LAST_OUTPUT" | perl -0777 -pe \
  's/.*?<promise>(.*?)<\/promise>.*/$1/s; s/^\s+|\s+$//g; s/\s+/ /g')

# 只有完全匹配才允许退出
if [[ "$PROMISE_TEXT" = "$COMPLETION_PROMISE" ]]; then
  rm "$RALPH_STATE_FILE"
  exit 0  # 允许退出
fi
```

Claude 的响应中必须包含完全匹配的 XML 标签，例如：
```
<promise>所有文档任务已完成</promise>
```

系统给 Claude 的约束规则：
```
CRITICAL RULE: If a completion promise is set, you may ONLY output it
when the statement is completely and unequivocally TRUE. Do not output
false promises to escape the loop, even if you think you're stuck.
```

### 4.4 /ralph-loop 与 Stop Hook 的协作时序

```
/ralph-loop 启动           Stop Hook 拦截            下次迭代
     │                          │                        │
     ▼                          │                        │
setup-ralph-loop.sh             │                        │
  创建状态文件                   │                        │
  iteration: 1                  │                        │
     │                          │                        │
     ▼                          │                        │
Claude 执行 Prompt               │                        │
     │                          │                        │
     ▼                          │                        │
Claude 完成，尝试结束 ────────────►│                        │
                            stop-hook.sh                 │
                            读取状态文件                  │
                            检查迭代/完成承诺              │
                                 │                        │
                            [未完成] ────────────────────►│
                                              iteration: 2
                                              喂回原始 Prompt
                                                   │
                                                   ▼
                                            Claude 再次执行
                                            （看到之前的文件）
```

### 4.5 取消机制

`/cancel-ralph` 命令提供手动退出路径：

```
1. 检查 .claude/ralph-loop.local.md 是否存在
2. 读取当前 iteration 值
3. 删除状态文件（下次 Stop 时 hook 检测不到文件，允许退出）
4. 报告："Cancelled Ralph loop (was at iteration N)"
```

### 4.6 应用场景与风险分析

**适合场景**：
- 需要多次迭代直到达标的任务（文档生成、代码重构直到测试通过）
- 自我改进循环（每次看到前次工作成果继续优化）
- 长时间无人值守的自动化任务

**设计风险**：
- **无限循环风险**：不设 `--max-iterations` 时，如果 Claude 无法真正完成任务会永久循环
- **完成承诺的诚实性**：依赖 Claude 不会在任务未完成时输出承诺文本
- **状态持久化**：状态文件在项目目录，多会话可能相互干扰

---

## 5. Plugin 开发八阶段工作流

**来源文件**：
- `plugins/plugin-dev/commands/create-plugin.md`
- `plugins/plugin-dev/agents/agent-creator.md`
- `plugins/plugin-dev/agents/plugin-validator.md`
- `plugins/plugin-dev/agents/skill-reviewer.md`

### 5.1 八阶段工作流总览

```
┌────────────────────────────────────────────────────────────────┐
│                  create-plugin 八阶段工作流                     │
└────────────────────────────────────────────────────────────────┘

Phase 1: Discovery（发现）
  理解插件目的 → 确认目标用户 → 类型分类

Phase 2: Component Planning（组件规划）
  ★ MUST 加载 plugin-structure Skill
  分析需求 → 确定组件类型 → 展示表格 → 用户确认

  表格示例：
  | 组件类型 | 数量 | 用途      |
  |----------|------|-----------|
  | Skills   | 2    | Hook模式  |
  | Commands | 3    | 部署/配置 |
  | Agents   | 1    | 自动验证  |
  | Hooks    | 0    | 不需要    |
  | MCP      | 1    | 数据库    |

Phase 3: Detailed Design & Clarifying Questions ★ CRITICAL
  对每个组件收集详细规格 → 等待用户回答

Phase 4: Plugin Structure Creation（结构创建）
  创建目录结构 → 写 plugin.json → README → .gitignore

Phase 5: Component Implementation（组件实现）
  ┌──────────────────────────────────────────────────────────┐
  │ 每种组件类型对应加载不同 Skill：                          │
  │ Skills   → skill-development Skill                       │
  │ Commands → command-development Skill                     │
  │ Agents   → agent-development Skill → 使用 agent-creator  │
  │ Hooks    → hook-development Skill                        │
  │ MCP      → mcp-integration Skill                         │
  │ Settings → plugin-settings Skill                         │
  └──────────────────────────────────────────────────────────┘

Phase 6: Validation & Quality Check（验证）
  ├── plugin-validator Agent → 全面验证
  ├── skill-reviewer Agent   → Skill 质量（如有）
  ├── validate-agent.sh      → Agent 验证（如有）
  └── validate-hook-schema.sh → Hook 验证（如有）

Phase 7: Testing & Verification（测试）
  提供安装指令 → 测试清单 → 指导用户测试

Phase 8: Documentation & Next Steps（文档）
  验证 README → Marketplace 入口 → 总结创建内容
```

### 5.2 关键决策点（用户确认节点）

工作流在 5 个关键节点必须获得用户确认：

```
Phase 1 结束 → 确认插件目的
     │
Phase 2 结束 → 批准组件计划
     │
Phase 3 结束 → 确认进入实现阶段
     │
Phase 6 结束 → 决定是否修复验证问题
     │
Phase 7 结束 → 继续到文档阶段
```

### 5.3 agent-creator Agent 的工作流程

agent-creator 本身就是一个用于创建 Agent 的 Agent，体现了系统的自我引用性：

```
1. 提取核心意图（目的/职责/成功标准）

2. 设计专家角色（expert persona）

3. 构建 Agent 配置：
   ├── identifier: 小写+连字符，3-50 字符
   ├── description: 包含触发条件 + 2-4 个 <example> 块
   ├── model: inherit（除非特殊需求）
   ├── color: 按职责选择（蓝/青=分析，绿=创建，黄=验证，红=安全）
   └── tools: 最小权限原则

4. 生成 agents/[identifier].md 文件

5. 建议运行 plugin-validator 验证
```

### 5.4 plugin-validator 的 10 步验证流程

```
Step 1:  定位插件根目录（.claude-plugin/plugin.json）
Step 2:  验证 Manifest（JSON 语法/必填字段/命名格式）
Step 3:  验证目录结构（commands/agents/skills/hooks）
Step 4:  验证 Commands（frontmatter/description/allowed-tools）
Step 5:  验证 Agents（name/description/model/color/system prompt）
Step 6:  验证 Skills（SKILL.md 存在/references 文件存在）
Step 7:  验证 Hooks（hooks.json 语法/事件名/脚本存在）
Step 8:  验证 MCP 配置（server type/连接方式）
Step 9:  检查文件组织（README/LICENSE/.gitignore）
Step 10: 安全检查（无硬编码凭证/HTTPS/无 secrets）
```

### 5.5 三 Agent 协作关系

```
┌─────────────────────────────────────────────────────────┐
│          plugin-dev 三 Agent 协作关系                    │
└─────────────────────────────────────────────────────────┘

           ┌──────────────────┐
           │  create-plugin   │  主命令（指挥调度）
           │     command      │
           └────────┬─────────┘
                    │ Phase 5（组件实现）
                    │ 对每个 Agent 类型
                    ▼
           ┌──────────────────┐
           │  agent-creator   │  创建 Agent 定义文件
           │  (magenta/sonnet)│  生成 identifier + examples + system prompt
           └────────┬─────────┘
                    │ 创建完成后
                    ▼
           ┌──────────────────┐
           │ plugin-validator │  验证整体插件结构
           │ (yellow/inherit) │  10 步全面验证，三态报告
           └────────┬─────────┘
                    │ 如有 Skill 组件
                    ▼
           ┌──────────────────┐
           │  skill-reviewer  │  专项 Skill 质量审查
           │  (cyan/inherit)  │  触发词/Progressive Disclosure/写作风格
           └──────────────────┘
```

---

## 6. 工作流模式横向对比

### 6.1 五大工作流核心特征对比

| 特征 | feature-dev | code-review | agent-sdk-dev | ralph-wiggum | plugin-dev |
|------|-------------|-------------|---------------|--------------|------------|
| **阶段数** | 7 | 9（步骤）| 5 | 无限/有限 | 8 |
| **用户确认节点** | 3 | 0（自动）| 5（一问一答）| 0（自动）| 5 |
| **并行 Agent** | 每阶段 2-3 | 4 个完全并行 | 无并行 | 无 Agent | 按需 |
| **置信度过滤** | ≥80 | 二次验证层 | 无 | 无 | 无 |
| **迭代机制** | 无 | 无 | 无 | Stop Hook 拦截 | 无 |
| **文档先行** | 无 | 无 | ✅（官方文档）| 无 | ✅（Skill 加载）|
| **自动验证** | 有（Phase 6）| 有（Step 5）| ✅（verifier Agent）| 无 | ✅（validator）|
| **主要模型策略** | sonnet 为主 | Haiku+Sonnet+Opus 混合 | Sonnet | N/A | Inherit |

### 6.2 「先问后做」程度比较

```
强←──────────────────────────────────────────────→弱

agent-sdk-dev    feature-dev    plugin-dev    code-review    ralph-wiggum
（一问一答式）    （3 个节点）   （5 个节点）   （全自动）     （全自动）
    ████████████   ██████████    ████████       ████           ██
```

- **agent-sdk-dev**：极强，每个问题独立问，不允许批量问答
- **feature-dev**：强，Phase 3 和 Phase 5 是必须等待的关卡
- **plugin-dev**：强，但允许在组件规划阶段批量展示并确认
- **code-review**：弱，完全自动，不等待用户
- **ralph-wiggum**：无，完全自主循环

### 6.3 Agent 数量与并行策略对比

| 工作流 | 最大并行 Agent 数 | 并行触发时机 | 结果聚合方式 |
|--------|-------------------|--------------|--------------|
| feature-dev | 3 | Phase 2/4/6 | 主 Agent 手动聚合 |
| code-review | 4+ | Step 4（核心阶段）| 主 Agent 过滤聚合 |
| pr-review-toolkit | 6 | 全部并行（可选）| 分级汇总（Critical/Important/Suggestions）|
| plugin-dev | 3 | Phase 6 验证 | 串行（validator→reviewer）|
| agent-sdk-dev | 1 | 最终验证 | 单 Agent 报告 |
| ralph-wiggum | 0 | N/A | N/A |

### 6.4 错误处理与健壮性策略对比

| 工作流 | 策略 |
|--------|------|
| code-review | 置信度 ≥80 + 独立 Agent 二次验证，双重防误报 |
| pr-review-toolkit | 每个专项 Agent 内部有独立标准，不相互干扰 |
| ralph-wiggum | 状态文件完整性检查 + 循环次数上限 + 完成承诺精确匹配 |
| agent-sdk-dev | TypeScript 零类型错误 + verifier Agent 验证 |
| feature-dev | 用户确认关卡 + Phase 6 多角度 review |

---

## 7. 关键发现总结

### 发现 1：分层置信度过滤是 AI Code Review 的核心设计

claude-code 的代码审查系统采用两种互补的过滤机制：
- **单 Agent 置信度**：code-reviewer 内部的 0-100 评分，≥80 才汇报
- **跨 Agent 二次验证**：code-review 插件在发现问题后启动独立 Agent 专门验证每个问题

两种机制解决了 AI 代码审查最核心的痛点：**误报率高**导致用户信任降低。

### 发现 2：「先探索后提问」是工作流的通用范式

无论是 feature-dev 还是 plugin-dev，都遵循相同的节奏：
1. 用 Agent 深度探索现有代码/规范
2. **基于探索结果**提出针对性问题
3. 获得用户回答后再行设计/实现

这与"立即开始实现"的对立面——先建立完整上下文，再针对具体情况提问，而非提通用模板问题。

### 发现 3：并行 Agent 的使用遵循"竞争探索，单一决策"原则

- **探索阶段**：2-3 个 Agent 并行，从不同角度探索（互补）
- **设计阶段**：2-3 个 Agent 并行提出不同方案（竞争）
- **审查阶段**：3 个 Agent 并行，从不同维度审查（分工）
- **决策/执行**：始终是主 Agent 或用户做单一决策

并行 Agent 的价值在于**视角多样性**，而非单纯的速度提升。

### 发现 4：Stop Hook 是实现自治 AI 循环的基础设施

Ralph Wiggum 将 Claude Code 的 Stop 事件 Hook 转化为"自我迭代"基础设施：
- 状态文件（`.claude/ralph-loop.local.md`）充当循环的持久化记忆
- Stop Hook 的 `decision: "block"` 机制实现了"退出拦截"
- 完成承诺（Completion Promise）通过 `<promise>` XML 标签精确匹配，防止 Claude 欺骗性退出

这是一种**利用现有 Hook 系统实现高级控制流**的创新设计模式。

### 发现 5：专项 Agent 的深度 > 通用 Agent 的广度

pr-review-toolkit 的 6 个专项 Agent 每个都有深刻的领域知识：
- `silent-failure-hunter`：有 5 个不可妥协原则，4 步系统化流程
- `type-design-analyzer`：有完整的四维度评分体系
- `comment-analyzer`：专注于"注释腐化"这一被忽视的技术债问题

这表明，对于代码审查类任务，**将领域专家的思维框架编码进 Agent 系统提示**比通用 AI 更可靠。

### 发现 6：工具权限的最小化原则

不同 Agent 的工具权限有明显区分：
- `code-explorer`：有 `BashOutput, KillShell`（需要执行脚本）
- `code-reviewer`：无写入工具（只读审查）
- `plugin-validator`：有 `Bash`（需要运行验证脚本）
- `skill-reviewer`：只有 `Read, Grep, Glob`（只读）

这种最小权限设计降低了 Agent 误操作的风险，同时通过工具列表明确表达了 Agent 的职责边界。

### 发现 7：工作流文档即提示词（Command = Prompt Engineering）

claude-code 的 command 文件本质上是精心工程化的提示词：
- 用 `## Phase N` 结构强制执行工作流顺序
- 用 `CRITICAL` / `DO NOT SKIP` 标注强制约束
- 用 `DO NOT START WITHOUT USER APPROVAL` 实现审批关卡
- 用 `**Ask user which approach they prefer**` 强制人机协作点

Command 文件不仅是文档，更是**强约束工作流的提示词模板**，让每次调用都遵循相同的最佳实践路径。

---

*文档生成时间：2026-04-08*  
*分析基于 `/Volumes/Data/agent-frameworks/claude-code/plugins/` 目录下的源文件*
