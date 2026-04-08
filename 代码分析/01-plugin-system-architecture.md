# Claude Code Plugin 系统架构深度分析

> 分析基于 `/Volumes/Data/agent-frameworks/claude-code/plugins/` 目录下的 13 个官方插件源码及相关文档。

---

## 目录

1. [Plugin 目录结构约定](#1-plugin-目录结构约定)
2. [Plugin 发现与加载机制](#2-plugin-发现与加载机制)
3. [Plugin 组件详细分析](#3-plugin-组件详细分析)
4. [官方插件横向对比](#4-官方插件横向对比)
5. [跨插件组件复用](#5-跨插件组件复用)
6. [关键发现总结](#6-关键发现总结)

---

## 1. Plugin 目录结构约定

### 1.1 标准目录布局

每个 Claude Code 插件遵循以下目录结构（来自 `plugin-dev/skills/plugin-structure/SKILL.md`）：

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 必需：插件元数据清单
├── commands/                 # 可选：斜线命令 (.md 文件)
├── agents/                   # 可选：子代理定义 (.md 文件)
├── skills/                   # 可选：技能目录
│   └── skill-name/
│       └── SKILL.md         # 每个技能必需此文件
├── hooks/
│   └── hooks.json           # 可选：事件钩子配置
├── .mcp.json                # 可选：MCP 服务器定义
├── scripts/                 # 可选：辅助脚本和工具
└── README.md                # 推荐：插件文档
```

### 1.2 各插件实际组件矩阵

通过扫描 13 个官方插件，汇总其实际使用的组件：

| 插件名 | `.claude-plugin/` | `commands/` | `agents/` | `skills/` | `hooks/` | Python 代码 | Shell 脚本 |
|--------|:-----------------:|:-----------:|:---------:|:---------:|:--------:|:-----------:|:----------:|
| agent-sdk-dev | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| claude-opus-4-5-migration | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| code-review | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| commit-commands | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| explanatory-output-style | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| feature-dev | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| frontend-design | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| hookify | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| learning-output-style | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| plugin-dev | ❌* | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| pr-review-toolkit | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| ralph-wiggum | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| security-guidance | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |

> *`plugin-dev` 插件的 `.claude-plugin/plugin.json` 在本地目录中缺失，但已注册于根目录的 `marketplace.json`。

**洞察/结论**：
- **唯一强制要求**：`.claude-plugin/plugin.json` 是插件被识别的唯一必需文件
- **commands、agents、skills、hooks 均是可选的**，插件可以只提供其中一种或多种组合
- **不同插件针对不同场景选择不同组件组合**，没有统一的"标准配置"

### 1.3 `plugin.json` 字段含义

**必需字段**（来自 `plugin-dev/skills/plugin-structure/references/manifest-reference.md`）：

```json
{
  "name": "plugin-name"
}
```

仅 `name` 字段是严格必需的，其余字段均可省略。

**完整字段说明**：

| 字段 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `name` | string | ✅ | kebab-case 格式唯一标识符，命名规则：`/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/` |
| `version` | string | ❌ | 语义版本（MAJOR.MINOR.PATCH），默认 `"0.1.0"` |
| `description` | string | ❌ | 50-200 字符说明，面向 Marketplace 展示 |
| `author` | object/string | ❌ | `{name, email, url}` 或字符串格式 |
| `homepage` | string | ❌ | 插件文档链接 |
| `repository` | string/object | ❌ | 源码仓库地址 |
| `license` | string | ❌ | SPDX 许可证标识符（如 `"MIT"`） |
| `keywords` | string[] | ❌ | 用于 Marketplace 搜索分类的标签 |
| `commands` | string/string[] | ❌ | 自定义命令目录路径（补充默认 `./commands`） |
| `agents` | string/string[] | ❌ | 自定义代理目录路径（补充默认 `./agents`） |
| `hooks` | string/object | ❌ | 钩子配置文件路径或内联配置 |
| `mcpServers` | string/object | ❌ | MCP 服务器配置文件路径或内联配置 |

**实际代码示例**（所有 13 个插件的 `plugin.json` 统一使用了 name/version/description/author 四个字段）：

```json
{
  "name": "hookify",
  "version": "0.1.0",
  "description": "Easily create hooks to prevent unwanted behaviors by analyzing conversation patterns",
  "author": {
    "name": "Daisy Hollman",
    "email": "daisy@anthropic.com"
  }
}
```

**洞察/结论**：
- 现有官方插件的 `plugin.json` 仅使用了规范定义的最小子集（name/version/description/author）
- 没有任何官方插件使用 `keywords`、`homepage`、`repository`、`license` 等扩展字段
- 没有任何官方插件在 `plugin.json` 中显式指定 `commands`/`agents`/`hooks` 路径——均依赖自动发现

---

## 2. Plugin 发现与加载机制

### 2.1 自动发现流程

Claude Code 通过以下顺序加载插件组件：

```
┌─────────────────────────────────────────────────────────┐
│                    插件启用流程                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. 读取 .claude-plugin/plugin.json                      │
│     └── 验证 name 字段格式（kebab-case）                 │
│     └── 验证 version 语义版本（如存在）                  │
│     └── 读取自定义路径配置                               │
│                                                          │
│  2. 扫描默认目录（自动发现）                             │
│     ├── commands/        → 加载所有 .md 文件             │
│     ├── agents/          → 加载所有 .md 文件             │
│     ├── skills/          → 加载含 SKILL.md 的子目录      │
│     ├── hooks/hooks.json → 加载钩子配置                  │
│     └── .mcp.json        → 加载 MCP 服务器               │
│                                                          │
│  3. 扫描 plugin.json 指定的自定义路径（补充合并）         │
│                                                          │
│  4. 注册所有发现的组件到 Claude Code 系统                 │
│                                                          │
│  关键：钩子在 Session Start 时加载，修改需重启           │
└─────────────────────────────────────────────────────────┘
```

### 2.2 `CLAUDE_PLUGIN_ROOT` 环境变量

这是 Plugin 系统中最核心的可移植性机制。在 `hookify/hooks/pretooluse.py` 中可以看到其完整用法：

```python
# hookify/hooks/pretooluse.py（第 14-23 行）
PLUGIN_ROOT = os.environ.get('CLAUDE_PLUGIN_ROOT')
if PLUGIN_ROOT:
    # 将插件父目录添加到 Python 路径，使 "hookify" 包可被导入
    parent_dir = os.path.dirname(PLUGIN_ROOT)
    if parent_dir not in sys.path:
        sys.path.insert(0, parent_dir)
    if PLUGIN_ROOT not in sys.path:
        sys.path.insert(0, PLUGIN_ROOT)
```

在 `hookify/hooks/hooks.json` 中，所有钩子命令均通过此变量引用路径：

```json
{
  "hooks": {
    "PreToolUse": [{
      "hooks": [{
        "type": "command",
        "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py",
        "timeout": 10
      }]
    }]
  }
}
```

**`CLAUDE_PLUGIN_ROOT` 作用机制**：

```
插件安装场景多样性
  ├── Marketplace 安装 → /Users/username/.claude/plugins/hookify/
  ├── 本地开发安装    → /path/to/my/plugins/hookify/
  ├── 项目级安装      → /project/.claude-plugins/hookify/
  └── 命令行指定      → claude --plugin-dir /custom/path/hookify/

所有场景下：
  CLAUDE_PLUGIN_ROOT = 插件根目录的绝对路径
  
因此 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py 
在所有安装场景下都能正确解析
```

**其他相关环境变量**（来自 `hook-development/SKILL.md`）：

| 变量 | 作用 |
|------|------|
| `$CLAUDE_PLUGIN_ROOT` | 当前插件的根目录绝对路径 |
| `$CLAUDE_PROJECT_DIR` | 用户正在工作的项目根目录 |
| `$CLAUDE_ENV_FILE` | 仅在 SessionStart 钩子中可用，用于向后续环境持久化环境变量 |
| `$CLAUDE_CODE_REMOTE` | 若在远程上下文中运行则被设置 |

**洞察/结论**：
- `CLAUDE_PLUGIN_ROOT` 是 Plugin 系统可移植性的核心设计，所有硬编码路径问题均通过它解决
- Python 的包导入机制需要特别处理：hookify 通过将 `parent_dir(PLUGIN_ROOT)` 加入 `sys.path` 来实现包导入
- Claude Code 在注册插件时注入此变量，插件作者无需关心实际安装位置

### 2.3 `marketplace.json` 的用途

根目录 `.claude-plugin/marketplace.json` 是整个仓库的插件目录索引：

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "version": "1.0.0",
  "owner": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  },
  "plugins": [
    {
      "name": "hookify",
      "description": "...",
      "version": "0.1.0",
      "source": "./plugins/hookify",
      "category": "productivity"
    }
    // ... 其他 12 个插件
  ]
}
```

**Marketplace 字段说明**：

| 字段 | 说明 |
|------|------|
| `$schema` | JSON Schema 验证地址，由 Anthropic 定义 |
| `name` | Marketplace 包名（不同于单个插件的 name） |
| `owner` | Marketplace 所有者（可以是组织） |
| `plugins[]` | 插件条目列表 |
| `plugins[].source` | 相对路径指向插件目录 |
| `plugins[].category` | 分类：development / productivity / learning / security |

**Marketplace 的角色**：

```
marketplace.json
    │
    ├── 作为"插件包"的索引，一个 marketplace 可以包含多个插件
    ├── 支持通过 /plugin 命令安装整个 marketplace 或其中特定插件
    ├── category 字段用于 Claude Code 插件市场的分类浏览
    └── source 字段允许从单个仓库分发多个插件
```

**洞察/结论**：
- `marketplace.json` 是"插件集合"级别的清单，而 `plugin.json` 是"单个插件"级别的清单
- 两者共存使得该仓库既可以作为整体 Marketplace 发布，又允许用户安装单个插件
- `category` 字段仅在 marketplace.json 中出现，不在单个 plugin.json 中，这是关注点分离的体现

---

## 3. Plugin 组件详细分析

### 3.1 Commands（斜线命令）

**文件格式**：Markdown 文件 + YAML Frontmatter

```
commands/
└── command-name.md    →   /command-name 或 /plugin-name:command-name
```

**Frontmatter 字段**（来自 `plugin-dev/skills/command-development/references/frontmatter-reference.md`）：

| 字段 | 必需 | 说明 |
|------|:----:|------|
| `description` | ❌ | `/help` 中显示的说明（≤60字符） |
| `allowed-tools` | ❌ | 限制可用工具（如 `["Bash", "Read"]`） |
| `model` | ❌ | 指定执行模型：`sonnet`/`opus`/`haiku` |
| `argument-hint` | ❌ | 参数提示（如 `[pr-number] [options]`） |
| `disable-model-invocation` | ❌ | `true` 则只允许用户手动调用，防止 Claude 自动调用 |
| `hide-from-slash-command-tool` | ❌ | `"true"` 则从斜线命令列表中隐藏 |

**命令体中的变量**：`$ARGUMENTS`（用户传入的参数）、`!` 前缀（执行 Bash 命令并内联结果）

```markdown
---
description: Guided feature development with codebase understanding
argument-hint: Optional feature description
---

# Feature Development
Initial request: $ARGUMENTS
```

**Ralph Wiggum 插件的高级用法**（`ralph-wiggum/commands/ralph-loop.md`）：

```markdown
---
description: "Start Ralph Wiggum loop in current session"
argument-hint: "PROMPT [--max-iterations N] [--completion-promise TEXT]"
allowed-tools: ["Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh:*)"]
hide-from-slash-command-tool: "true"
---
```

注意：`allowed-tools` 中使用了 `${CLAUDE_PLUGIN_ROOT}` 路径过滤器，将 Bash 工具限制为只能运行特定脚本。

### 3.2 Agents（子代理）

**文件格式**：Markdown 文件 + YAML Frontmatter

```
agents/
└── agent-name.md    →   可被 Task 工具调用的子代理
```

**触发方式**：
- Claude Code 根据任务上下文自动匹配（基于 `description` 字段）
- 用户可以在对话中显式提及代理名
- 父命令可以通过 `Task` 工具显式调用

**`feature-dev` 插件代理协作模式**：

```
/feature-dev 命令
    │
    ├── Phase 2：并行启动 2-3 个 code-explorer 代理
    │   └── 每个代理负责不同维度的代码探索
    │
    ├── Phase 4：并行启动 2-3 个 code-architect 代理
    │   └── 生成不同权衡的架构方案
    │
    └── Phase 6：并行启动 3 个 code-reviewer 代理
        └── 分别检查：简洁性、正确性、约定遵循
```

### 3.3 Skills（技能）

**文件结构**：

```
skills/
└── skill-name/
    ├── SKILL.md              # 必需：技能核心文档
    ├── references/           # 可选：详细参考资料
    │   └── detail-topic.md
    ├── examples/             # 可选：工作示例
    │   └── example-code.sh
    └── scripts/              # 可选：工具脚本
        └── utility.sh
```

**SKILL.md Frontmatter**：

```markdown
---
name: Hook Development
description: This skill should be used when the user asks to "create a hook"...
version: 0.1.0
---
```

**触发机制**：Claude Code 根据 `description` 字段中的关键词和用户请求自动匹配并激活技能。触发词越具体，匹配越准确。

**渐进式信息披露（Progressive Disclosure）设计模式**：

```
SKILL.md（精简核心，≈1500-2000词）
    ├── 快速参考（Quick Reference）
    ├── 关键模式示例
    └── → 指向 references/ 获取完整细节

references/（详细技术文档）
    ├── patterns.md      → 完整模式库
    ├── advanced.md      → 高级用法
    └── migration.md     → 迁移指南

examples/（工作代码示例）
    ├── validate-write.sh
    └── load-context.sh
```

**洞察/结论**：SKILL.md 的设计哲学是"精简核心 + 外部引用"，避免单文件过大。`plugin-dev` 插件的 7 个 Skill 是目前最复杂的技能集合，每个技能都有完整的 references/ 和 examples/ 子结构。

### 3.4 Hooks（事件钩子）

**支持的事件类型**（来自 `hook-development/SKILL.md`）：

| 事件 | 触发时机 | 典型用途 |
|------|----------|----------|
| `PreToolUse` | 工具执行前 | 验证、拦截、修改工具调用 |
| `PostToolUse` | 工具执行后 | 反馈、日志记录 |
| `Stop` | 主代理准备停止时 | 验证任务完整性 |
| `SubagentStop` | 子代理停止时 | 子任务验证 |
| `UserPromptSubmit` | 用户提交 Prompt 时 | 上下文注入、验证 |
| `SessionStart` | 会话开始时 | 环境配置、上下文加载 |
| `SessionEnd` | 会话结束时 | 清理、日志 |
| `PreCompact` | 上下文压缩前 | 保留关键信息 |
| `Notification` | Claude 发送通知时 | 日志、响应 |

**钩子类型**：

```json
// Prompt-based Hook（推荐：利用 LLM 进行上下文感知决策）
{
  "type": "prompt",
  "prompt": "验证此工具调用是否安全...",
  "timeout": 30
}

// Command Hook（命令式：确定性检查）
{
  "type": "command",
  "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/check.py",
  "timeout": 10
}
```

**hooks.json 的两种格式**：

```json
// Plugin 格式（有 "hooks" 包装层）
{
  "description": "钩子说明",
  "hooks": {
    "PreToolUse": [...]
  }
}

// Settings 格式（直接使用，无包装层）
{
  "PreToolUse": [...]
}
```

**钩子输出格式**（根据事件类型不同而不同）：

```json
// PreToolUse 钩子输出（许可决策）
{
  "hookSpecificOutput": {
    "permissionDecision": "allow|deny|ask",
    "updatedInput": {"field": "modified_value"}
  },
  "systemMessage": "给 Claude 的说明"
}

// Stop 钩子输出（继续/阻止决策）
{
  "decision": "approve|block",
  "reason": "原因说明"
}
```

**并行执行**：所有匹配同一事件的钩子**并行执行**，钩子之间无法互相访问对方的输出。

**退出码语义**：
- `0`：成功（stdout 显示在 transcript 中）
- `2`：阻断错误（stderr 反馈给 Claude）
- 其他：非阻断错误

---

## 4. 官方插件横向对比

### 4.1 设计模式分类

#### 模式一：纯命令工作流（command-centric workflow）

**代表插件**：`commit-commands`、`code-review`

```
commit-commands/
├── .claude-plugin/plugin.json
├── commands/
│   ├── commit.md           → /commit
│   ├── commit-push-pr.md   → /commit-push-pr
│   └── clean_gone.md       → /clean_gone
└── README.md
```

**设计哲学**：最简化原则。提供三个独立的 git 工作流命令，每个命令是一个完整的独立工作单元，无需代理或技能支持。

**适用场景**：重复性的、步骤固定的操作，用户明确知道何时调用。

---

#### 模式二：命令 + 代理协作（command + multi-agent orchestration）

**代表插件**：`feature-dev`、`pr-review-toolkit`

```
feature-dev/
├── .claude-plugin/plugin.json
├── commands/
│   └── feature-dev.md      → 7 阶段工作流编排器
├── agents/
│   ├── code-explorer.md    → 并行探索子代理
│   ├── code-architect.md   → 并行架构子代理
│   └── code-reviewer.md    → 并行评审子代理
└── README.md
```

**设计哲学**：命令作为"工作流编排器"，代理作为"专业执行单元"。命令定义阶段流程和检查点，代理专注于特定维度的深度分析。

`pr-review-toolkit` 将此模式进一步细化：

```
pr-review-toolkit/
└── agents/
    ├── code-reviewer.md        → 通用代码质量
    ├── comment-analyzer.md     → 注释准确性
    ├── pr-test-analyzer.md     → 测试覆盖率
    ├── silent-failure-hunter.md → 静默失败检测
    ├── type-design-analyzer.md  → 类型设计
    └── code-simplifier.md      → 代码简化
```

每个代理都有明确的专业边界（confidence-based filtering），避免误报。

**适用场景**：复杂的多维度分析任务，需要并行处理且各维度相互独立。

---

#### 模式三：纯技能注入（skill-only）

**代表插件**：`frontend-design`、`claude-opus-4-5-migration`

```
frontend-design/
├── .claude-plugin/plugin.json
└── skills/
    └── frontend-design/
        └── SKILL.md
```

**设计哲学**：最小干预原则。不提供命令或代理，仅通过技能向 Claude 的行为注入领域知识和最佳实践。技能会在 Claude 处理相关任务时自动激活，无需用户显式调用。

**适用场景**：跨项目的通用最佳实践，希望对用户透明地改善 AI 的输出质量。

---

#### 模式四：事件驱动自动化（hook-driven automation）

**代表插件**：`security-guidance`、`explanatory-output-style`、`learning-output-style`

```
security-guidance/
├── .claude-plugin/plugin.json
└── hooks/
    ├── hooks.json                  → PreToolUse 钩子配置
    └── security_reminder_hook.py   → 安全检查逻辑
```

**设计哲学**：被动监控原则。无命令，无代理，无技能——纯粹通过钩子在特定事件发生时自动介入。`security-guidance` 在文件写入前扫描 9 种安全模式；`explanatory-output-style` 在 Session 开始时注入教育性提示。

`security-guidance` 钩子的精妙设计——会话级去重：

```python
# security_reminder_hook.py（第 260-273 行）
warning_key = f"{file_path}-{rule_name}"
shown_warnings = load_state(session_id)

if warning_key not in shown_warnings:
    shown_warnings.add(warning_key)
    save_state(session_id, shown_warnings)
    print(reminder, file=sys.stderr)
    sys.exit(2)  # 仅在首次触发时阻断
```

**适用场景**：安全策略执行、合规检查、教育性提示——不需要用户主动调用，而是在后台默默工作。

---

#### 模式五：可配置规则引擎（configurable rule engine）

**代表插件**：`hookify`

```
hookify/
├── .claude-plugin/plugin.json
├── commands/
│   ├── hookify.md          → 创建新规则
│   ├── list.md             → 列出规则
│   ├── configure.md        → 配置规则
│   └── help.md             → 帮助
├── agents/
│   └── conversation-analyzer.md  → 分析对话模式
├── skills/
│   └── writing-rules/
│       └── SKILL.md        → 规则编写指导
├── hooks/
│   ├── hooks.json          → 四类事件钩子
│   ├── pretooluse.py
│   ├── posttooluse.py
│   ├── stop.py
│   └── userpromptsubmit.py
└── core/                   → Python 引擎库
    ├── config_loader.py    → YAML Frontmatter 解析器
    └── rule_engine.py      → 规则评估引擎
```

**设计哲学**：元插件原则。hookify 本身是一个 Plugin，但它的核心价值是让用户无需编写代码就能创建自定义钩子。用户通过 Markdown 文件配置规则：

```markdown
---
name: dangerous-rm
enabled: true
event: bash
pattern: "rm -rf"
action: warn
---

⚠️ Dangerous rm -rf command detected!
```

规则文件存储在 `.claude/hookify.*.local.md`，由 `config_loader.py` 在运行时加载并解析。

**规则引擎架构**：

```
.claude/hookify.*.local.md （用户规则文件）
    │
    ▼
config_loader.py
    ├── extract_frontmatter() → 解析 YAML frontmatter
    ├── Rule.from_dict()      → 创建规则对象
    └── load_rules(event)     → 按事件过滤并返回规则列表
    │
    ▼
rule_engine.py
    ├── RuleEngine.evaluate_rules() → 评估所有规则
    ├── _rule_matches()             → 检查工具匹配
    ├── _check_condition()          → 检查条件满足
    └── _extract_field()            → 从 Hook 输入提取字段值
    │
    ▼
输出格式根据事件类型：
    ├── PreToolUse → {hookSpecificOutput: {permissionDecision}}
    ├── Stop       → {decision: "block", reason: "..."}
    └── 其他       → {systemMessage: "..."}
```

**适用场景**：需要高度可定制化行为控制，且希望让非开发者也能配置规则的场景。

---

#### 模式六：元开发工具包（meta-development toolkit）

**代表插件**：`plugin-dev`

```
plugin-dev/
├── commands/
│   └── create-plugin.md    → 8 阶段创建向导
├── agents/
│   ├── agent-creator.md    → AI 辅助创建代理
│   ├── plugin-validator.md → 验证器代理
│   └── skill-reviewer.md   → 技能评审代理
└── skills/
    ├── plugin-structure/   → 插件架构知识
    ├── hook-development/   → 钩子开发知识
    ├── agent-development/  → 代理开发知识
    ├── command-development/→ 命令开发知识
    ├── skill-development/  → 技能开发知识
    ├── mcp-integration/    → MCP 集成知识
    └── plugin-settings/    → 配置管理知识
```

**设计哲学**：自我参照原则。`plugin-dev` 使用 Plugin 系统的所有特性来教导用户如何构建 Plugin，形成一个完美的自我参照系统。7 个专业技能覆盖 Plugin 开发的每一个方面。

---

#### 模式七：自我迭代循环（self-referential loop）

**代表插件**：`ralph-wiggum`

```
ralph-wiggum/
├── commands/
│   ├── ralph-loop.md       → 启动循环
│   └── cancel-ralph.md     → 取消循环
├── hooks/
│   ├── hooks.json          → Stop 钩子
│   └── stop-hook.sh        → 拦截退出，重新注入 Prompt
└── scripts/
    └── setup-ralph-loop.sh → 初始化循环环境
```

**设计哲学**：持续性迭代原则。通过拦截 `Stop` 事件，将同一任务反复注入 Claude，使其在看到前次工作成果的基础上不断改进，直到满足完成条件。

```
/ralph-loop TASK [--completion-promise TEXT]
    │
    ▼
claude 执行任务 → 尝试停止
    │                │
    │                ▼
    │         Stop Hook 拦截
    │                │
    │                ▼
    │         检查 completion-promise 是否满足
    │         ├── 满足 → 允许停止
    └─────────└── 不满足 → 重新注入 TASK，继续迭代
```

**适用场景**：需要多次迭代才能完成的复杂任务，或者需要 Claude 自我验证结果的场景。

---

### 4.2 各插件设计哲学对比

| 插件 | 主要模式 | 核心哲学 | 交互方式 |
|------|----------|----------|----------|
| commit-commands | 纯命令 | 最小化重复操作 | 用户主动调用 |
| feature-dev | 命令 + 代理 | 结构化工作流 + 专业分工 | 用户主动，多阶段引导 |
| pr-review-toolkit | 命令 + 代理 | 多维度专业评审 | 用户主动，并行分析 |
| frontend-design | 纯技能 | 透明质量提升 | 自动激活，用户无感 |
| claude-opus-4-5-migration | 纯技能 | 领域迁移知识注入 | 自动激活 |
| hookify | 规则引擎 | 用户可配置的行为控制 | 混合（配置 + 自动执行） |
| security-guidance | 纯钩子 | 被动安全守卫 | 完全自动，后台运行 |
| ralph-wiggum | 循环机制 | 自我迭代完善 | 用户启动，自动迭代 |
| plugin-dev | 元工具包 | 自举式知识系统 | 用户主动，引导式创建 |
| agent-sdk-dev | 命令 + 代理 | SDK 项目脚手架 | 用户主动，交互式 |

---

## 5. 跨插件组件复用

### 5.1 技能共享机制

Claude Code 的技能共享不通过插件间直接引用实现，而通过**缓存安装**机制：

```
用户安装的插件 A（含 skill-x）→ ~/.cursor/plugins/cache/plugin-a/skills/skill-x/
用户安装的插件 B（含 skill-y）→ ~/.cursor/plugins/cache/plugin-b/skills/skill-y/

所有已安装插件的技能均对 Claude Code 可见（只要插件已启用）
```

在 Cursor 的 Agent Skills 系统中（`.cursor/projects/.../agent-skills` 路径），可以看到不同插件的技能文件被统一列出，供 Claude 选择调用。

**实际观察到的复用模式**：`plugin-dev` 插件的技能内容已被复制到 Cursor 插件缓存，并作为独立技能对 Agent 可见：
- `plugin-dev/skills/hook-development/SKILL.md` → Cursor 系统可调用
- `plugin-dev/skills/agent-development/SKILL.md` → Cursor 系统可调用

### 5.2 代理复用

**代理名冲突问题**：由于不同插件可能定义相同名称的代理（如 `feature-dev` 和 `pr-review-toolkit` 都有 `code-reviewer` 代理），Claude Code 通过命名空间解决冲突：

```
/feature-dev:code-reviewer       → feature-dev 插件的 code-reviewer
/pr-review-toolkit:code-reviewer → pr-review-toolkit 插件的 code-reviewer
```

### 5.3 插件设置（Plugin Settings）机制

`plugin-settings` Skill 定义了一种跨插件的配置约定：

```
.claude/
└── hookify.local.md           # 格式：{plugin-name}.local.md
```

文件使用 YAML Frontmatter + Markdown 内容格式：

```markdown
---
# YAML 配置部分
api_endpoint: https://example.com
max_results: 50
---

# Markdown 说明部分（可供人类阅读）
This file configures the hookify plugin...
```

**已在 `.gitignore` 中排除的约定**：用户配置文件 `.claude/*.local.md` 默认不应该提交到版本控制，因为它们通常包含用户特定的配置或敏感信息。

### 5.4 安装与项目级配置

```
项目级配置层级：
├── ~/.claude/settings.json     → 全局用户设置（所有项目共享）
├── .claude/settings.json       → 项目级设置
├── .claude/settings.local.json → 项目级本地设置（不提交）
└── .claude/*.local.md          → 插件特定配置（不提交）

插件安装方式：
├── /plugin install <marketplace-url>  → 从 Marketplace 安装
├── cc --plugin-dir /path/to/plugin    → 命令行指定插件目录
└── .claude/settings.json 中配置       → 项目内引用
```

---

## 6. 关键发现总结

### 发现 1：Plugin 系统是"约定优于配置"的典型实现

Claude Code 的 Plugin 系统遵循"Convention Over Configuration"原则：只需将文件放在正确的位置，无需任何配置即可自动发现和加载。`.claude-plugin/plugin.json` 中仅需 `name` 字段就能创建一个可工作的插件。

### 发现 2：`${CLAUDE_PLUGIN_ROOT}` 是跨安装场景可移植性的关键

所有涉及文件路径的钩子和 MCP 配置都**必须**使用 `${CLAUDE_PLUGIN_ROOT}` 而非硬编码路径。hookify 的 Python 代码更进一步，通过将该变量作为 `sys.path` 条目来支持 Python 包导入。

### 发现 3：Plugin 系统支持两种截然不同的干预模式

- **主动模式**（commands/agents）：用户显式调用，Claude 按需执行
- **被动模式**（hooks/skills）：系统自动触发，对用户透明

`security-guidance` 和 `explanatory-output-style` 是纯被动模式的典型代表，安装即生效，无需用户操作。

### 发现 4：`hooks.json` 有两种不同格式，对应不同使用场景

Plugin 内的 `hooks/hooks.json` 需要 `{"hooks": {...}}` 包装层，而用户 `settings.json` 中的钩子配置直接使用事件名。这是一个微妙的格式差异，初学者容易混淆。

### 发现 5：代理并行执行是 Plugin 工作流的核心性能模式

`feature-dev` 和 `pr-review-toolkit` 大量使用并行代理模式：同时启动多个专业代理，每个代理聚焦于不同维度，汇总后呈现给用户。这比串行执行大幅减少等待时间。

### 发现 6：技能的"渐进式披露"设计是文档工程的最佳实践

技能文件保持精简（≤2000 词），通过 `references/` 和 `examples/` 子目录提供详细内容。这使得技能核心内容易于 AI 快速解析，同时保留了完整的参考资料。

### 发现 7：hookify 是一个"元插件"——用 Plugin 系统构建配置 Plugin 的工具

hookify 最具创新性：它将 Plugin 的 Hook 机制暴露给最终用户，让用户无需编写代码就能定制 Claude Code 的行为。其规则引擎（`rule_engine.py` + `config_loader.py`）是生产级 Python 实现，支持正则匹配、条件组合、阻断/警告两种动作。

### 发现 8：Marketplace 是插件分发的标准化机制

根目录的 `marketplace.json` 定义了一个标准的"插件包"格式，允许单个 Git 仓库作为多个插件的分发源。`category` 字段是 Marketplace 特有的，用于分类浏览，不同于插件自身的 `keywords`。

### 发现 9：ralph-wiggum 通过 Stop 钩子实现了一种新的执行范式

通过拦截 Claude 的"停止意图"并重新注入相同任务，ralph-wiggum 创造了一种持续迭代的执行循环。这是 Plugin 钩子系统能力边界的极限探索——将"被动事件响应"用于"主动控制执行流程"。

### 发现 10：Plugin 系统与 Claude Code Settings 形成两层配置体系

- **Plugin 层**：`hooks/hooks.json`（包装格式），随插件安装自动注册
- **Settings 层**：`.claude/settings.json`（直接格式），用户手动配置

两层同时生效，Plugin 的钩子与用户的钩子**合并**运行（不互相覆盖），允许用户在插件行为之上进行个性化叠加。

---

*文档生成时间：2026-04-08*  
*分析范围：`/Volumes/Data/agent-frameworks/claude-code/plugins/` 目录下 13 个官方插件*
