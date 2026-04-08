# Claude Code Skills（技能库）系统深度分析

> 分析范围：`/Volumes/Data/agent-frameworks/claude-code/plugins/` 下所有 Skills 相关目录  
> 分析日期：2026-04-08

---

## 目录

1. [SKILL.md 格式规范](#1-skillmd-格式规范)
2. [Skill 文件组织结构](#2-skill-文件组织结构)
3. [Skill 触发与调用机制](#3-skill-触发与调用机制)
4. [Skills 在复杂工作流中的角色](#4-skills-在复杂工作流中的角色)
5. [Skills vs Agents vs Commands 三者对比](#5-skills-vs-agents-vs-commands-三者对比)
6. [各 Skill 全量对比表](#6-各-skill-全量对比表)
7. [关键发现总结](#7-关键发现总结)

---

## 1. SKILL.md 格式规范

### 1.1 基础结构

每个 SKILL.md 由两部分构成：**YAML frontmatter**（元数据）+ **Markdown 正文**（指令内容）。

```
skill-name/
└── SKILL.md
    ├── --- (YAML frontmatter) ---
    │   ├── name: (必须)       技能名称
    │   ├── description: (必须) 触发描述，包含具体触发短语
    │   └── version: (可选)    版本号，如 0.1.0
    └── Markdown 正文 (指令内容)
        ├── ## Overview          概述
        ├── ## 核心内容章节      主体知识
        ├── ## Quick Reference   快速参考
        └── ## Additional Resources  扩展资源指引
```

### 1.2 Frontmatter 字段规范

| 字段 | 是否必须 | 规范 | 示例 |
|------|---------|------|------|
| `name` | 必须 | 技能展示名称，标题格式 | `Hook Development` |
| `description` | 必须 | 第三人称，含具体触发短语 | `This skill should be used when...` |
| `version` | 可选 | semver 格式 | `0.1.0` |
| `license` | 可选 | 商业/开源授权说明 | `Complete terms in LICENSE.txt` |

#### Description 字段的严格规范

描述字段是 Skill 触发机制的核心，须满足：

1. **第三人称格式**：`This skill should be used when the user asks to...`（禁止 `Use this skill when...`）
2. **包含具体触发短语**：用引号括起用户可能说的原话
3. **场景描述**：说明适用情境而非功能描述

**正确示例（hook-development）：**
```yaml
description: This skill should be used when the user asks to "create a hook", 
  "add a PreToolUse/PostToolUse/Stop hook", "validate tool use", 
  "implement prompt-based hooks", "use ${CLAUDE_PLUGIN_ROOT}", 
  "set up event-driven automation", "block dangerous commands", or mentions 
  hook events (PreToolUse, PostToolUse, Stop, SubagentStop, SessionStart, 
  SessionEnd, UserPromptSubmit, PreCompact, Notification).
```

**错误示例：**
```yaml
description: Use this skill when working with hooks.  # 错：非第三人称，无触发短语
description: Provides hook guidance.                   # 错：纯功能描述
```

### 1.3 Markdown 正文写作规范

| 规范 | 要求 | 说明 |
|------|------|------|
| **语气** | 命令/不定式形式 | 用 `To create a hook, define...` 而非 `You should create...` |
| **篇幅** | 1,500-2,000 词 | 理想范围，最多 5,000 词 |
| **结构** | 概述 → 核心内容 → 快速参考 → 扩展资源 | 遵循渐进式披露设计 |
| **人称** | 禁止第二人称 | 禁止 `You should...`、`You need to...` |

### 1.4 渐进式披露设计（Progressive Disclosure）

Skills 系统采用三层上下文加载机制，以高效管理 LLM 上下文窗口：

```
第1层：元数据（始终在上下文中）
  └── name + description ≈ 100词
      作用：让 Claude 知道该 Skill 存在，决定是否调用

第2层：SKILL.md 正文（技能触发时加载）
  └── 核心流程 + 快速参考 + 扩展资源指引 < 5,000词
      作用：提供完成任务所需的核心知识和工作流

第3层：捆绑资源（按需加载，无限制）
  └── references/   → 详细文档（各文件 2,000-5,000 词）
  └── examples/     → 完整可运行代码示例
  └── scripts/      → 可执行脚本（可直接运行，无需加载到上下文）
  └── assets/       → 输出用资产（模板、图片等）
```

此设计的核心优势：**Skill 越复杂，越需要把详细内容移入 references/，保持 SKILL.md 精简**。

---

## 2. Skill 文件组织结构

### 2.1 完整目录结构（以 hook-development 为例）

```
plugin-dev/skills/hook-development/
├── SKILL.md                          # 必须：核心文档（约1,651词）
│
├── examples/                         # 工作示例（直接可用代码）
│   ├── validate-write.sh             # 文件写入验证示例
│   ├── validate-bash.sh              # Bash 命令验证示例
│   └── load-context.sh               # SessionStart 上下文加载示例
│
├── references/                       # 详细参考文档（按需加载）
│   ├── patterns.md                   # 8+种常见 Hook 模式（含代码）
│   ├── advanced.md                   # 高级用法（多阶段验证、跨事件工作流等）
│   └── migration.md                  # 从基础 Hook 迁移到高级 Hook 的指南
│
└── scripts/                          # 开发辅助工具脚本
    ├── README.md                     # 脚本说明文档
    ├── validate-hook-schema.sh       # 验证 hooks.json 结构与语法
    ├── test-hook.sh                  # 在部署前用样本输入测试 Hook
    └── hook-linter.sh                # 检查 Hook 脚本的常见问题
```

### 2.2 各目录的分工模式

```
examples/   ← 「直接可用」：完整可运行的代码文件，用户可直接复制和调整
references/ ← 「按需阅读」：详细文档，Claude 判断需要时才加载入上下文
scripts/    ← 「工具执行」：辅助开发的脚本，可直接运行无需读入上下文
assets/     ← 「输出资产」：模板、图片等，用于最终输出，不加载到上下文
```

### 2.3 六大 Skill 的目录结构对比

```
plugin-dev/skills/
│
├── hook-development/           ★ 最完整的示例
│   ├── SKILL.md
│   ├── examples/               ✓（3个示例脚本）
│   ├── references/             ✓（3个文档：patterns, advanced, migration）
│   └── scripts/                ✓（3个工具：validate, test, lint）
│
├── agent-development/
│   ├── SKILL.md
│   ├── examples/               ✓（2个：agent-creation-prompt, complete-agent-examples）
│   ├── references/             ✓（3个：system-prompt-design, triggering-examples, agent-creation-system-prompt）
│   └── scripts/                ✓（2个：validate-agent, test-agent-trigger）
│
├── command-development/
│   ├── SKILL.md
│   ├── examples/               ✓（2个：plugin-commands, simple-commands）
│   └── references/             ✓（6个：advanced-workflows, documentation-patterns, 
│                                       frontmatter-reference, interactive-commands, 
│                                       marketplace-considerations, plugin-features-reference,
│                                       testing-strategies）
│
├── mcp-integration/
│   ├── SKILL.md
│   ├── examples/               ✓（3个：stdio, sse, http 配置示例）
│   └── references/             ✓（3个：server-types, authentication, tool-usage）
│
├── plugin-settings/
│   ├── SKILL.md
│   ├── examples/               ✓（3个：read-settings-hook, create-settings-command, example-settings）
│   ├── references/             ✓（2个：parsing-techniques, real-world-examples）
│   └── scripts/                ✓（2个：validate-settings, parse-frontmatter）
│
├── plugin-structure/
│   ├── SKILL.md
│   ├── examples/               ✓（3个：minimal-plugin, standard-plugin, advanced-plugin）
│   └── references/             ✓（2个：component-patterns, manifest-reference）
│
└── skill-development/
    ├── SKILL.md
    └── references/             ✓（1个：skill-creator-original）
```

### 2.4 references/ 下各文档的分工模式

以 `hook-development/references/` 为例：

| 文件 | 定位 | 内容规模 | 加载时机 |
|------|------|---------|---------|
| `patterns.md` | 常见模式合集 | 10个模式 + 代码 | 需要实现具体 Hook 时 |
| `advanced.md` | 高级技术 | 多阶段验证、外部集成等 | 遇到复杂需求时 |
| `migration.md` | 迁移指南 | 从旧式到新式的转换 | 升级现有 Hook 时 |

**分工原则**：SKILL.md 负责「应该做什么」，references/ 负责「具体怎么做」。

### 2.5 scripts/ 目录的用途深度分析

`scripts/` 与 `examples/` 的根本区别：

| 维度 | scripts/ | examples/ |
|------|---------|---------|
| **目标用户** | 开发者（构建插件时） | Claude（执行任务时） |
| **执行方式** | 直接命令行运行 | 作为参考代码 |
| **内容形态** | 工具/辅助脚本 | 完整功能示例 |
| **典型用途** | 验证、测试、Lint | 可复制的代码模板 |
| **调用时机** | 开发阶段 | 理解/参考阶段 |

具体工具示例（hook-development）：
- `validate-hook-schema.sh`：检查 hooks.json 的语法、必填字段、超时值范围
- `test-hook.sh`：模拟 Claude Code 环境，用样本 JSON 输入测试 Hook 脚本
- `hook-linter.sh`：检测常见安全问题（未引用变量、注入风险、硬编码路径）

---

## 3. Skill 触发与调用机制

### 3.1 触发机制工作原理

```
用户发送消息
      │
      ▼
Claude 读取所有 Skill 的 name + description（始终在上下文）
      │
      ▼
匹配 description 中的触发条件
      │
   ┌──┴──┐
   │命中？│
   └──┬──┘
  是  │  否
      │    └──→ 忽略该 Skill
      ▼
加载 SKILL.md 正文
      │
      ▼
按照 SKILL.md 指引执行任务
（按需读取 references/、examples/、scripts/）
```

### 3.2 触发短语设计分析

**触发短语的层次结构**（以 hook-development 为例）：

```
精确动词短语（最强触发信号）
  "create a hook"
  "add a PreToolUse/PostToolUse/Stop hook"
  "validate tool use"
  "implement prompt-based hooks"

技术术语（中等触发信号）
  "${CLAUDE_PLUGIN_ROOT}"
  "set up event-driven automation"
  "block dangerous commands"

概念关键词（弱触发信号，但已足够）
  hook events (PreToolUse, PostToolUse, Stop, ...)
```

### 3.3 各 Skill 的触发设计特点

#### plugin-settings（最具特色的隐式触发）
```yaml
description: This skill should be used when the user asks about "plugin settings", 
  "store plugin configuration", "user-configurable plugin", ".local.md files", 
  "plugin state files", "read YAML frontmatter", "per-project plugin settings"
```
特点：通过`.local.md files`这类技术文件名触发，而非通用概念。

#### frontend-design（最简洁的描述式触发）
```yaml
description: Create distinctive, production-grade frontend interfaces with high 
  design quality. Use this skill when the user asks to build web components, 
  pages, or applications.
```
特点：采用「描述+场景」触发，无需引号短语，触发条件更宽泛。注意违反了「第三人称」规范（实际用了 `Use this skill when` 而非 `This skill should be used when`），但因场景明确仍有效。

#### claude-opus-4-5-migration（最窄域的精确触发）
```yaml
description: Migrate prompts and code from Claude Sonnet 4.0, Sonnet 4.5, or 
  Opus 4.1 to Opus 4.5. Use when the user wants to update their codebase, 
  prompts, or API calls to use Opus 4.5.
```
特点：以版本号为核心触发，极度精准，不会误触发。

#### writing-rules（hookify 插件内的嵌套 Skill）
```yaml
description: This skill should be used when the user asks to "create a hookify rule", 
  "write a hook rule", "configure hookify", "add a hookify rule"
```
特点：触发短语包含插件名 `hookify`，形成命名空间隔离，避免与通用 hook 概念混淆。

### 3.4 自动触发的元描述设计

Skill 的 `description` 本质上是 **触发元提示（Trigger Meta-Prompt）**，Claude 通过对话意图匹配来激活技能。设计原则：

1. **具体 > 抽象**：`"create a hook"` > `"hook-related task"`
2. **动词引导**：短语以动词开头（create, add, write, configure）
3. **包含同义词**：覆盖同一意图的多种表达
4. **技术术语补充**：当用户使用技术词汇（如 `PreToolUse`）时也能触发
5. **否定范围**：有时需说明 Skill 不适用的情况（如 `Does NOT migrate Haiku 4.5`）

---

## 4. Skills 在复杂工作流中的角色

### 4.1 feature-dev 工作流中 Skills 与 Agents 的协作模式

`feature-dev` 插件没有 Skills，只有 Agents + Commands，体现了两种模式的不同适用场景：

```
feature-dev 工作流：
  /feature-dev (Command) ← 用户触发
        │
        ├──→ code-explorer (Agent) ← 探索代码库
        ├──→ code-architect (Agent) ← 设计架构方案
        └──→ code-reviewer (Agent) ← 审查实现代码
```

如果 feature-dev 增加 Skills，会是这样的协作：

```
/feature-dev (Command)
        │
        ├──→ code-explorer (Agent)
        │         └── 使用 「api-testing Skill」（技术知识）
        │
        ├──→ code-architect (Agent)
        │         └── 使用 「design-patterns Skill」（领域知识）
        │
        └──→ code-reviewer (Agent)
                  └── 使用 「code-standards Skill」（规范知识）
```

**核心协作模式**：Skills 为 Agents 提供专业知识基础，Agents 负责执行决策，Commands 负责触发工作流。

### 4.2 plugin-dev 工作流中 Skills 的角色

`plugin-dev` 是同时拥有 Skills、Agents、Commands 的最完整插件，展示了三者完整的协作方式：

```
plugin-dev 插件架构：

Commands（用户触发）
  └── /create-plugin

Agents（自动触发，复杂任务）
  ├── agent-creator      ← 创建新 Agent
  ├── plugin-validator   ← 验证插件结构
  └── skill-reviewer     ← 审查 Skill 质量

Skills（知识库，自动注入）
  ├── hook-development   ← "create a hook" 时加载
  ├── command-development ← "create a command" 时加载
  ├── agent-development  ← "create an agent" 时加载
  ├── mcp-integration    ← "add MCP server" 时加载
  ├── plugin-settings    ← ".local.md files" 时加载
  ├── plugin-structure   ← "create a plugin" 时加载
  └── skill-development  ← "create a skill" 时加载
```

**Skills 在此的关键作用**：
- `agent-creator` Agent 创建 Agent 时，`agent-development` Skill 自动提供规范知识
- `plugin-validator` Agent 验证插件时，`plugin-structure` Skill 提供结构规范
- `skill-reviewer` Agent 审查 Skill 质量时，`skill-development` Skill 提供质量标准

### 4.3 hookify 工作流：Skills 与 Hooks 的协作

`hookify` 插件独特之处在于它的 Skill 辅助其自身的 Hook 系统运作：

```
hookify 工作流：

用户想配置 hookify 规则
        │
        ▼
writing-rules Skill 激活（"create a hookify rule" 触发）
        │
        ▼
Skill 教导如何写 .claude/hookify.*.local.md 规则文件
        │
        ▼
hookify 的 Hook 系统读取这些规则文件并执行
        │
        ├── PreToolUse ← 拦截危险命令
        ├── PostToolUse ← 检查代码质量  
        └── Stop ← 验证完成标准
```

**设计亮点**：`writing-rules` Skill 是 hookify 系统的「配置向导」，降低了用户使用 Hook 系统的学习成本。

### 4.4 claude-opus-4-5-migration：迁移指导 Skill 的设计模式

此 Skill 展示了「一次性任务型 Skill」的设计模式：

```
迁移工作流（由 Skill 驱动）：

触发：用户提到 "migrate to Opus 4.5"
      │
      ▼
SKILL.md 加载（定义完整迁移流程）：
  步骤1：搜索所有模型字符串
  步骤2：按平台替换为目标版本
  步骤3：移除不支持的 beta 头部
  步骤4：添加 effort 参数
  步骤5：汇总所有更改
      │
      ├── references/effort.md ← 按需加载（effort 参数详情）
      └── references/prompt-snippets.md ← 按需加载（提示词调整片段）
```

**设计亮点**：
1. Skill 本身就是完整的操作手册，无需外部 Agent 协调
2. 通过 references/ 分离了「核心流程」和「参考数据」（模型字符串、提示词片段）
3. 明确的「仅在用户报告问题时应用提示词调整」降低了过度优化的风险

---

## 5. Skills vs Agents vs Commands 三者对比

### 5.1 核心区别对比表

| 维度 | Skills（技能） | Agents（代理） | Commands（命令） |
|------|--------------|--------------|----------------|
| **本质** | 知识与规范的注入 | 自主执行复杂任务的子进程 | 用户触发的预定义提示 |
| **触发方式** | 自动（意图匹配） | 自动（描述匹配）或手动 | 手动（用户输入 `/命令名`） |
| **执行模式** | 无独立进程，注入当前上下文 | 独立子进程，有自己的系统提示 | 在当前会话中扩展为指令 |
| **适用场景** | 专业领域知识、重复任务规范 | 多步骤自主工作、复杂分析 | 频繁使用的标准工作流 |
| **文件格式** | `SKILL.md`（YAML+Markdown） | `agent-name.md`（YAML+Markdown） | `command-name.md`（YAML+Markdown） |
| **必须字段** | `name`, `description` | `name`, `description`, `model`, `color` | 无（纯内容即可） |
| **存储位置** | `skills/<skill-name>/SKILL.md` | `agents/<agent-name>.md` | `commands/<command-name>.md` |
| **资源捆绑** | ✅（references/examples/scripts/assets） | ❌（无捆绑资源） | ❌（无捆绑资源） |
| **工具访问** | 继承当前上下文 | 可限制（`tools: [...]`） | 可限制（`allowed-tools: ...`） |
| **系统提示** | 无（注入指令） | 有（定义身份和行为） | 无（成为用户指令） |
| **持久状态** | ❌ | ❌（任务完成即结束） | ❌ |
| **模型选择** | 继承当前 | 可指定（`model: haiku/sonnet/opus`） | 可指定（`model: ...`） |
| **颜色标识** | 无 | 有（`color: blue/green/red...`） | 无 |

### 5.2 适用场景对比

| 适用场景 | 推荐组件 | 理由 |
|---------|---------|------|
| 教导 Claude 如何创建 Hook | **Skill** | 知识注入，自动触发 |
| 自动分析 PR 代码质量 | **Agent** | 需要自主决策和多步执行 |
| 快速启动代码审查流程 | **Command** | 用户频繁手动调用 |
| 提供 API 文档参考 | **Skill**（references/） | 知识库，按需加载 |
| 监听并阻止危险文件写入 | **Hook** | 事件驱动，自动执行 |
| 配置 hookify 规则的向导 | **Skill** | 辅助配置，知识注入 |
| 迁移代码到新模型版本 | **Skill** | 操作规范，一次性任务 |
| 验证插件结构是否合法 | **Agent** | 分析任务，需要工具使用 |

### 5.3 写作人称对比

| 组件 | Frontmatter description | 正文内容 |
|------|------------------------|---------|
| **Skill** | 第三人称：`This skill should be used when...` | 命令/不定式：`To create X, do Y` |
| **Agent** | 第三人称（带示例）：`Use this agent when... <example>...</example>` | 第二人称：`You are an agent that...` |
| **Command** | 第一行（如有描述字段） | 为 Claude 的指令，动词开头 |

---

## 6. 各 Skill 全量对比表

| Skill 名称 | 所在插件 | 触发场景核心 | 是否有 references | 是否有 examples | 是否有 scripts | 正文词数（约） |
|-----------|---------|------------|-----------------|----------------|--------------|-------------|
| **Hook Development** | plugin-dev | Hook 创建与配置 | ✅ 3个 | ✅ 3个 | ✅ 3个 | ~1,651 |
| **Command Development** | plugin-dev | Slash 命令开发 | ✅ 6个 | ✅ 2个 | ❌ | ~1,800 |
| **Agent Development** | plugin-dev | Agent 创建与设计 | ✅ 3个 | ✅ 2个 | ✅ 2个 | ~1,438 |
| **MCP Integration** | plugin-dev | MCP 服务器集成 | ✅ 3个 | ✅ 3个 | ❌ | ~1,600 |
| **Plugin Settings** | plugin-dev | 插件配置文件管理 | ✅ 2个 | ✅ 3个 | ✅ 2个 | ~1,500 |
| **Plugin Structure** | plugin-dev | 插件目录结构 | ✅ 2个 | ✅ 3个 | ❌ | ~1,400 |
| **Skill Development** | plugin-dev | Skill 本身的创建 | ✅ 1个 | ❌ | ❌ | ~2,200 |
| **Writing Hookify Rules** | hookify | hookify 规则语法 | ❌ | ❌（内嵌） | ❌ | ~800 |
| **frontend-design** | frontend-design | 前端界面创建 | ❌ | ❌ | ❌ | ~450 |
| **claude-opus-4-5-migration** | claude-opus-4-5-migration | Claude 模型版本迁移 | ✅ 2个 | ❌ | ❌ | ~600 |

---

## 7. 关键发现总结

### 发现 1：Skills 系统是「知识经济」而非「执行系统」

Skills 不执行代码，不调用工具，不产生副作用——它们是**专业知识的结构化注入机制**。当 Claude 遇到特定领域任务时，Skill 自动激活并为其注入该领域的「领域专家知识」。这与 Agents（执行者）和 Commands（触发器）形成了功能互补的三角关系。

### 发现 2：渐进式披露是 Skills 系统最重要的设计哲学

从 100 词的描述，到 2,000 词的 SKILL.md，再到无限制的 references/，Skills 系统通过分层加载避免了上下文窗口浪费。**越核心的知识越靠前，越详细的内容越靠后**。这种设计使得 Skills 可以既保持触发的精准性，又不损失知识的完整性。

### 发现 3：Description 是 Skills 系统的灵魂

`description` 字段决定了 Skill 能否被正确触发，是整个系统中最关键的设计决策。最佳实践证明：**具体的动词短语（"create a hook"）> 技术术语（PreToolUse）> 抽象描述**。第三人称格式不只是风格规范，而是让 Claude 能将描述作为「他人的能力说明」而非「自我指令」来理解。

### 发现 4：hook-development 是 Skills 系统的最佳范例

`hook-development` Skill 是该系统中资源最完整的实现，完美体现了「最小核心文档 + 按需详细资源 + 开发工具链」的设计模式：
- SKILL.md 精简（~1,651词）但完整覆盖核心概念
- `references/` 分为「模式」「高级」「迁移」三个关注点独立文件
- `scripts/` 提供了 validate/test/lint 完整的开发工具链
- `examples/` 提供了三种最常见场景的完整实现代码

### 发现 5：writing-rules Skill 揭示了「插件内聚」设计模式

`hookify` 插件的 `writing-rules` Skill 是为插件自身功能服务的——它教用户如何配置 hookify，而 hookify 的 Hooks 则执行这些配置。这揭示了一个重要模式：**Skill 可以成为插件功能的「交互式文档」**，将用户引导手册内化为自动触发的知识注入，大幅降低用户学习成本。

### 发现 6：Skill 与 Agent 存在明显的协作分工边界

通过分析 plugin-dev 插件，可以看到清晰的分工边界：
- `skill-reviewer` **Agent** 负责「审查 Skill 是否符合规范」（执行决策）
- `skill-development` **Skill** 负责「教导如何创建优质 Skill」（知识注入）

这种「Agent 判断，Skill 指导」的模式在所有插件中保持一致，形成了 Claude Code 插件生态的核心架构范式。

### 发现 7：frontend-design 和 claude-opus-4-5-migration 展示了两种极端 Skill 形态

- **frontend-design**：纯创意指南型 Skill，无 references/examples/scripts，直接在 SKILL.md 中内嵌所有知识，适合无需结构化数据的创意类任务
- **claude-opus-4-5-migration**：精准操作手册型 Skill，SKILL.md 是步骤清单，references/ 是数据文件（模型字符串、提示词片段），适合需要精确查表的迁移类任务

这两种形态代表了 Skills 系统在「软知识」和「硬数据」两端的不同表达方式。

---

*文档生成自对 `/Volumes/Data/agent-frameworks/claude-code/plugins/` 目录的深度静态分析。*
