# Claude Code 代码探索 TODO

> **状态：✅ 全部完成（2026-04-08）**
>
> 目标：从多个维度系统分析 Claude Code 的插件生态、扩展机制、工作流设计和底层原理，建立完整的认知地图。
>
> 说明：Claude Code 的核心 CLI 二进制是闭源的，本仓库重点包含：插件系统示例、Hooks 机制、自定义命令、Agent 定义、MCP 集成配置等可探索内容。

---

## 一、Plugin 系统架构

> 目标：理解 Claude Code 插件的完整生命周期、目录约定和元数据格式。

- [x] **1.1 Plugin 目录结构约定**
  - [x] 分析 `plugins/` 下所有插件的目录布局（commands / agents / skills / hooks / .claude-plugin）
  - [x] 对比各插件之间的差异，归纳哪些部分是必需的，哪些是可选的
  - [x] 梳理 `plugin.json` 的字段含义（name、version、description、author）

- [x] **1.2 Plugin 发现与加载机制**
  - [x] 研究 `.claude-plugin/plugin.json` 如何被 Claude Code 发现和加载
  - [x] 分析 `CLAUDE_PLUGIN_ROOT` 环境变量的作用与传递方式
  - [x] 了解 `marketplace.json`（`.claude-plugin/marketplace.json`）的用途

- [x] **1.3 Plugin 之间的组件复用**
  - [x] 研究跨插件的 skill 共享方式（skills 目录引用机制）
  - [x] 了解 plugin 安装与项目级配置的关系（`/plugin` 命令行为）

- [x] **1.4 官方插件横向对比**
  - [x] 对比 `hookify`、`feature-dev`、`plugin-dev`、`pr-review-toolkit` 的设计模式
  - [x] 总结各插件的适用场景和设计哲学差异

---

## 二、Hooks 系统

> 目标：完整掌握 Claude Code 的事件驱动扩展机制，理解每种 Hook 的触发时机、输入输出协议和退出码语义。

- [x] **2.1 Hook 事件类型全景**
  - [x] 梳理所有 Hook 事件：`PreToolUse`、`PostToolUse`、`Stop`、`UserPromptSubmit`、`SessionStart`、`SessionEnd`、`SubagentStop`、`PreCompact`、`Notification`
  - [x] 建立事件触发时序图：Session 生命周期 → 每次工具调用 → 用户提交 → 停止

- [x] **2.2 Hook 输入/输出协议（stdin/stdout/stderr）**
  - [x] 分析 stdin JSON 的结构（`tool_name`、`tool_input`、`hook_event_name` 等字段）
  - [x] 研究 stdout 响应格式（`systemMessage`、`hookSpecificOutput`、`stopReason`）
  - [x] 掌握退出码语义：0（继续）、1（stderr 显示给用户但不阻断）、2（阻断并反馈给 Claude）

- [x] **2.3 Hookify 插件深度分析**
  - [x] 阅读 `hookify/hooks/pretooluse.py` 完整实现，理解规则加载与评估流程
  - [x] 阅读 `hookify/core/rule_engine.py` —— 规则引擎的正则匹配、lru_cache 缓存策略
  - [x] 阅读 `hookify/core/config_loader.py` —— `.local.md` 规则文件的解析方式
  - [x] 分析 `hookify/hooks/stop.py`、`posttooluse.py`、`userpromptsubmit.py` 的差异

- [x] **2.4 Hook 配置方式**
  - [x] 研究 `hooks.json` 配置格式（matcher、type、command、timeout）
  - [x] 对比 `hooks.json` 与 `settings.json` 中 hooks 配置的区别
  - [x] 分析 `matcher` 字段的作用（工具名匹配、通配符）

- [x] **2.5 安全 Hook 实现案例**
  - [x] 分析 `security-guidance` 插件的 PreToolUse hook 实现（多种危险模式检测）
  - [x] 分析 `examples/hooks/bash_command_validator_example.py` —— grep→rg 的替换拦截逻辑
  - [x] 研究 `ralph-wiggum` 的 Stop hook 如何拦截退出并实现迭代循环

- [x] **2.6 SessionStart Hook 注入上下文**
  - [x] 分析 `explanatory-output-style` 的 SessionStart hook 如何注入教育性提示
  - [x] 分析 `learning-output-style` 的 SessionStart hook 的设计差异

---

## 三、Commands（自定义斜杠命令）系统

> 目标：掌握自定义命令的定义格式、参数传递机制和复杂工作流编排。

- [x] **3.1 Command Frontmatter 规范**
  - [x] 分析 `commands/*.md` 文件的 YAML frontmatter 字段（description、argument-hint）
  - [x] 研究 `$ARGUMENTS` 占位符的使用与多参数传递方式
  - [x] 了解命令文件命名与斜杠命令映射关系（`feature-dev.md` → `/feature-dev`）

- [x] **3.2 多阶段工作流命令设计**
  - [x] 深度阅读 `feature-dev/commands/feature-dev.md` —— 7 阶段工作流的完整设计
  - [x] 分析 `plugin-dev/commands/create-plugin.md` —— 8 阶段插件创建引导
  - [x] 研究命令中如何启动并行 Agent（Phase 2 的 2-3 个并行 explorer 模式）

- [x] **3.3 交互式命令设计模式**
  - [x] 研究命令中"等待用户回答再继续"的设计约定
  - [x] 分析 `hookify/commands/hookify.md`、`configure.md`、`list.md`、`help.md` 的命令集合模式
  - [x] 了解命名空间命令约定（`/hookify:configure` 等子命令格式）

- [x] **3.4 项目级命令与全局命令**
  - [x] 分析 `.claude/commands/` 目录下项目级命令的作用（`triage-issue.md`、`commit-push-pr.md`）
  - [x] 研究 `dedupe.md` 的实现逻辑（自动去重命令）

---

## 四、Agents（专用智能体）系统

> 目标：理解 Agent 的定义格式、工具权限控制、模型选择和触发条件设计。

- [x] **4.1 Agent Frontmatter 格式**
  - [x] 分析 Agent `.md` 文件的 frontmatter 字段：`name`、`description`、`model`、`color`、`tools`
  - [x] 研究 `model: inherit` vs `model: sonnet` vs `model: opus` 的选择策略
  - [x] 了解 `color` 字段的作用（UI 区分标识）

- [x] **4.2 工具权限控制**
  - [x] 分析不同 Agent 的 `tools` 列表设计（`code-explorer` vs `conversation-analyzer`）
  - [x] 研究最小权限原则在 Agent 工具分配中的体现
  - [x] 了解 Agent 无法使用的工具类型（限制原因）

- [x] **4.3 Agent 触发条件设计**
  - [x] 深度分析 `description` 字段中 `<example>` 标签的结构和 `<commentary>` 的作用
  - [x] 研究触发条件的精确性设计——如何避免误触发
  - [x] 对比 `conversation-analyzer`、`code-explorer`、`code-architect`、`code-reviewer` 的触发场景

- [x] **4.4 多 Agent 并行编排**
  - [x] 分析 `feature-dev` 命令中如何同时启动多个 code-explorer 的并行模式
  - [x] 研究 `pr-review-toolkit` 中 5 个并行评审 Agent 的分工与汇总机制
  - [x] 了解 Agent 结果如何回传给主 Agent 或命令流程

- [x] **4.5 Agent 系统提示设计**
  - [x] 深度阅读 `code-explorer.md`、`code-architect.md`、`code-reviewer.md` 的系统提示
  - [x] 分析高质量 Agent 系统提示的结构（Core Mission → Analysis Approach → Output Guidance）
  - [x] 研究 `plugin-dev` 中 `agent-creator.md`、`plugin-validator.md`、`skill-reviewer.md` 的定义风格

---

## 五、Skills（技能库）系统

> 目标：理解 Skills 的结构设计、渐进式披露模式和与 Agent/Command 的协作机制。

- [x] **5.1 SKILL.md 格式规范**
  - [x] 分析 `skills/*/SKILL.md` 的结构（description、触发条件、核心指令）
  - [x] 研究 Skills 的渐进式披露设计（概述 → 详细引用 → examples/references）
  - [x] 理解 Skills 与 Agents/Commands 的根本区别（技能是指导，Agent 是执行者）

- [x] **5.2 Skill 文件组织结构**
  - [x] 分析 `plugin-dev/skills/hook-development/` 的完整目录（SKILL.md + examples + references + scripts）
  - [x] 研究 `skills/command-development/references/` 下各参考文档的分工
  - [x] 了解 `scripts/` 目录在 Skill 中的用途（`hook-linter.sh`、`test-hook.sh`）

- [x] **5.3 Skill 触发与调用机制**
  - [x] 研究 Skill 如何被 Agent 或用户指令触发
  - [x] 分析 `hookify/skills/writing-rules/SKILL.md` 的设计——如何指导规则编写
  - [x] 了解 `frontend-design` 的 Skill 如何实现"自动触发"的元描述

- [x] **5.4 Skills 在复杂工作流中的角色**
  - [x] 分析 `feature-dev` 工作流中 Skills 与 Agents 的协作模式
  - [x] 研究 `claude-opus-4-5-migration` Skill 的自动化迁移指导设计

---

## 六、MCP（Model Context Protocol）集成

> 目标：理解 Claude Code 如何通过 MCP 连接外部工具和服务。

- [x] **6.1 .mcp.json 配置格式**
  - [x] 分析 `plugin-dev/skills/mcp-integration/examples/` 下三种配置示例（stdio、SSE、HTTP）
  - [x] 研究 `.mcp.json` 的字段结构（mcpServers、transport、command、args、env）
  - [x] 了解项目级 `.mcp.json` 与全局 MCP 配置的优先级关系

- [x] **6.2 MCP Server 类型对比**
  - [x] 阅读 `references/server-types.md`——stdio vs SSE vs HTTP vs WebSocket 的适用场景
  - [x] 分析 `examples/stdio-server.json` 的本地命令行工具集成模式
  - [x] 分析 `examples/sse-server.json` 的远程事件流服务集成模式

- [x] **6.3 MCP 与 Plugin 的协作**
  - [x] 研究 MCP 工具如何在 Agent 的 `tools` 列表中暴露
  - [x] 了解 `references/authentication.md` 中 MCP 认证的方案

---

## 七、Settings 与权限体系

> 目标：完整掌握 Claude Code 的配置分层、权限控制和沙箱机制。

- [x] **7.1 settings.json 配置层级**
  - [x] 分析 `examples/settings/` 下三种预设（settings-lax.json、settings-strict.json、settings-bash-sandbox.json）
  - [x] 研究配置的优先级顺序（全局 → 项目级 → 命令行参数）
  - [x] 梳理 `permissions` 对象的字段（`disableBypassPermissionsMode`、`strictKnownMarketplaces`）

- [x] **7.2 .local.md 插件设置文件**
  - [x] 分析 `hookify/examples/*.local.md` 文件的 YAML frontmatter 结构
  - [x] 研究 `.local.md` 的命名约定（`.claude/hookify.*.local.md`）和加载机制
  - [x] 对比 `console-log-warning.local.md`、`dangerous-rm.local.md`、`require-tests-stop.local.md` 的规则设计

- [x] **7.3 Bash 沙箱与权限控制**
  - [x] 深度分析 `settings-bash-sandbox.json` 的沙箱配置
  - [x] 研究 `devcontainer/init-firewall.sh` 的防火墙规则（网络隔离实现）
  - [x] 了解 `devcontainer.json` 中容器级权限控制的设计

---

## 八、核心工作流设计模式

> 目标：从宏观视角理解 Claude Code 推崇的开发工作流和 AI 协作模式。

- [x] **8.1 Feature Development 七阶段工作流**
  - [x] 完整阅读 `feature-dev/commands/feature-dev.md` 所有阶段（Discovery → Exploration → Clarification → Architecture → Implementation → Testing → Review）
  - [x] 分析每个阶段的 Agent 组合策略（并行 explorer → 串行 clarify → 并行 architect）
  - [x] 研究"先问后做"原则在工作流中的落地方式

- [x] **8.2 Code Review 并行 Agent 模式**
  - [x] 分析 `code-review` 插件的 5 并行 Agent 设计（CLAUDE.md 合规、Bug 检测、历史上下文、PR 历史、代码注释）
  - [x] 研究置信度过滤机制如何减少误报
  - [x] 了解 `pr-review-toolkit` 的 6 个专项 Agent 的分工（comments、tests、errors、types、code、simplify）

- [x] **8.3 Agent SDK 开发工作流**
  - [x] 分析 `agent-sdk-dev` 插件的 `/new-sdk-app` 命令——SDK 项目交互式初始化流程
  - [x] 研究 `agent-sdk-verifier-py` 和 `agent-sdk-verifier-ts` 的验证机制

- [x] **8.4 Ralph Wiggum 自我迭代循环**
  - [x] 深度分析 `ralph-wiggum` 的 Stop hook 拦截 + `/ralph-loop` 命令实现的自驱迭代机制
  - [x] 研究这种"自主循环"模式的应用场景与风险

- [x] **8.5 Plugin 开发八阶段工作流**
  - [x] 完整阅读 `plugin-dev/commands/create-plugin.md`——引导式插件创建流程
  - [x] 分析 `agent-creator`、`plugin-validator`、`skill-reviewer` 三个 Agent 的协作关系

---

## 九、CI/CD 与 GitHub 集成

> 目标：理解 Claude Code 的 GitHub Actions 自动化设计，了解 AI 如何介入 Issue 管理和 PR 审查。

- [x] **9.1 claude.yml —— Claude 介入 GitHub Actions**
  - [x] 分析 `.github/workflows/claude.yml` 的触发条件和执行逻辑
  - [x] 了解 Claude 如何响应 GitHub Issue 和 PR 的评论

- [x] **9.2 Issue 生命周期自动化**
  - [x] 分析 `claude-issue-triage.yml` —— AI 自动分类 Issue 的流程
  - [x] 研究 `auto-close-duplicates.yml` + `scripts/auto-close-duplicates.ts`——重复 Issue 检测与关闭
  - [x] 了解 `issue-lifecycle-comment.yml` + `scripts/lifecycle-comment.ts` 的 Issue 状态通知机制

- [x] **9.3 脚本实现分析**
  - [x] 阅读 `scripts/sweep.ts` —— 批量 Issue 清理逻辑
  - [x] 分析 `scripts/gh.sh` —— GitHub CLI 封装脚本的用途
  - [x] 研究 `scripts/backfill-duplicate-comments.ts` 的设计意图

---

## 十、Claude Code 行为与设计哲学

> 目标：从用户视角理解 Claude Code 的设计哲学、使用模式和扩展边界。

- [x] **10.1 Claude Code 整体架构理解**
  - [x] 阅读 `README.md` 和 `CHANGELOG.md`——了解产品定位和演化历程
  - [x] 研究 Claude Code 作为"agentic coding tool"的核心能力边界
  - [x] 了解 npm 安装（deprecated）vs 官方安装方式的变化原因

- [x] **10.2 安全性设计哲学**
  - [x] 分析 `SECURITY.md` 的安全声明和漏洞报告流程
  - [x] 研究 `security-guidance` 插件覆盖的危险模式检测（command injection、XSS、eval、os.system 等）
  - [x] 了解 devcontainer + 防火墙的沙箱隔离思路

- [x] **10.3 数据采集与隐私**
  - [x] 分析 `README.md` 中数据采集范围（usage data、conversation data、bug feedback）
  - [x] 了解 Claude Code 的数据保留策略和训练数据使用限制

- [x] **10.4 Plugin 生态扩展模式**
  - [x] 研究 Marketplace 的设计（strictKnownMarketplaces 配置含义）
  - [x] 了解社区 Plugin 的发布和安装方式
  - [x] 分析"输出风格 Plugin"（explanatory-output-style、learning-output-style）的设计思路——通过 SessionStart Hook 注入行为

---

## 探索优先级建议

| 优先级 | 主题 | 理由 |
|--------|------|------|
| P0 | Hooks 系统（主题二） | 最核心的扩展机制，理解输入输出协议是一切的基础 |
| P0 | Plugin 架构（主题一） | 理解整体组织结构，是后续所有探索的前提 |
| P1 | Commands 系统（主题三） | 最直接的用户入口，设计模式丰富 |
| P1 | Agents 系统（主题四） | 多 Agent 并行是 Claude Code 最有特色的能力 |
| P2 | Skills 系统（主题五） | 理解 Skills 的角色定位 |
| P2 | Settings/权限（主题七） | 安全模型理解 |
| P3 | MCP 集成（主题六） | 外部工具接入 |
| P3 | 工作流设计（主题八） | 从宏观理解最佳实践 |
| P4 | CI/CD 集成（主题九） | GitHub Actions 自动化 |
| P4 | 行为与哲学（主题十） | 产品理解 |

---

*生成时间：2026-04-08*
*仓库位置：`/Volumes/Data/agent-frameworks/claude-code`*

---

## 产出文档索引

| 文件 | 对应主题 | 状态 |
|------|----------|------|
| [01-plugin-system-architecture.md](./01-plugin-system-architecture.md) | 一、Plugin 系统架构 | ✅ 完成 |
| [02-hooks-system.md](./02-hooks-system.md) | 二、Hooks 系统 | ✅ 完成 |
| [03-commands-system.md](./03-commands-system.md) | 三、Commands 系统 | ✅ 完成 |
| [04-agents-system.md](./04-agents-system.md) | 四、Agents 系统 | ✅ 完成 |
| [05-skills-system.md](./05-skills-system.md) | 五、Skills 系统 | ✅ 完成 |
| [06-mcp-integration.md](./06-mcp-integration.md) | 六、MCP 集成 | ✅ 完成 |
| [07-settings-permissions.md](./07-settings-permissions.md) | 七、Settings 与权限体系 | ✅ 完成 |
| [08-workflow-patterns.md](./08-workflow-patterns.md) | 八、核心工作流设计模式 | ✅ 完成 |
| [09-cicd-github-integration.md](./09-cicd-github-integration.md) | 九、CI/CD 与 GitHub 集成 | ✅ 完成 |
| [10-design-philosophy.md](./10-design-philosophy.md) | 十、Claude Code 行为与设计哲学 | ✅ 完成 |

*完成时间：2026-04-08*
