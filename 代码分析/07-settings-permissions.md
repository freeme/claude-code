# Claude Code Settings 与权限体系深度分析

> **文档版本**：2026-04-08  
> **分析范围**：`examples/settings/`、`plugins/hookify/examples/`、`.devcontainer/`、`.claude-plugin/marketplace.json`

---

## 目录

1. [权限层级架构总览](#1-权限层级架构总览)
2. [settings.json 配置体系](#2-settingsjson-配置体系)
3. [.local.md 插件设置文件](#3-localmd-插件设置文件)
4. [Bash 沙箱与网络隔离](#4-bash-沙箱与网络隔离)
5. [Plugin Marketplace 信任机制](#5-plugin-marketplace-信任机制)
6. [权限设计哲学](#6-权限设计哲学)
7. [关键发现总结](#7-关键发现总结)

---

## 1. 权限层级架构总览

Claude Code 的权限体系采用多层防御设计，从基础设施到应用配置形成完整的安全纵深：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code 权限体系层级                       │
├─────────────────────────────────────────────────────────────────┤
│  Layer 0: 基础设施层（DevContainer）                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Docker 容器隔离 (NET_ADMIN + NET_RAW caps)              │    │
│  │  iptables 防火墙（白名单出站规则，默认 DROP）              │    │
│  │  ipset 域名白名单（GitHub / npm / Anthropic API 等）      │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: 企业托管设置层（managed-settings.json）                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  allowManagedPermissionRulesOnly: true   ← 锁定权限规则   │    │
│  │  allowManagedHooksOnly: true             ← 锁定 Hooks    │    │
│  │  strictKnownMarketplaces: [...]          ← 白名单市场     │    │
│  │  permissions.disableBypassPermissionsMode ← 禁用跳过模式  │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: 应用权限层（settings.json）                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  permissions.allow / ask / deny          ← 工具白名单    │    │
│  │  sandbox.enabled / network              ← Bash 沙箱     │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: 插件钩子层（hookify .local.md rules）                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PreToolUse hooks   → 命令/文件操作前拦截                 │    │
│  │  PostToolUse hooks  → 操作后审计                          │    │
│  │  Stop hooks         → 会话结束前强制检查                  │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: 运行时模式层                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  --dangerously-skip-permissions  ← 可被上层禁用           │    │
│  │  bypass permissions mode UI     ← VSCode 权限选择器      │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**关键原则**：越高层的设置具有更高优先级，企业托管设置 (`managed-settings.json`) 可覆盖用户/项目级设置，且某些属性只有在企业设置层才生效。

---

## 2. settings.json 配置体系

### 2.1 三种预设配置对比

Claude Code 官方提供三种预设配置，覆盖从最宽松到最严格的使用场景：

| 配置项 | `settings-lax.json`<br>（宽松模式） | `settings-strict.json`<br>（严格模式） | `settings-bash-sandbox.json`<br>（沙箱模式） |
|--------|:---:|:---:|:---:|
| **禁用 `--dangerously-skip-permissions`** | ✅ | ✅ | ❌ |
| **屏蔽第三方 Plugin 市场** | ✅（空白名单） | ✅（空白名单） | ❌ |
| **禁止用户/项目自定义权限规则** | ❌ | ✅ | ✅ |
| **禁止用户/项目自定义 Hooks** | ❌ | ✅ | ❌ |
| **拒绝 Web 搜索/抓取工具** | ❌ | ✅（deny） | ❌ |
| **Bash 工具需手动审批** | ❌ | ✅（ask） | ❌ |
| **Bash 工具强制在沙箱内运行** | ❌ | ❌ | ✅ |
| **沙箱自动允许 Bash** | ❌ | ❌（显式禁用） | ❌（显式禁用） |
| **允许沙箱内 Unix Sockets** | ❌ | ❌ | ❌ |
| **允许本地端口绑定** | ❌ | ❌ | ❌ |
| **弱化嵌套沙箱** | ❌ | ❌ | ❌ |

**适用场景**：
- `settings-lax.json`：**合规基线**，适用于已有外部安全控制的企业环境，仅阻止最危险操作（绕过权限 + 不受控插件市场）。
- `settings-strict.json`：**高安全要求**，适用于处理敏感数据、需要严格审计的团队，Bash 必须每次手动审批，网络访问受限。
- `settings-bash-sandbox.json`：**沙箱隔离**，适用于 CI/CD 或不信任代码执行环境，所有 Bash 命令必须在操作系统级沙箱内运行。

### 2.2 核心配置字段详解

#### `permissions` 对象

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "allow": ["ToolName"],
    "ask": ["Bash"],
    "deny": ["WebSearch", "WebFetch"]
  }
}
```

| 字段 | 类型 | 含义 |
|------|------|------|
| `disableBypassPermissionsMode` | `"disable"` \| `undefined` | 设为 `"disable"` 时，禁用 `--dangerously-skip-permissions` 启动参数，同时在 VSCode UI 中隐藏权限跳过选项（v2.1.70+） |
| `allow` | `string[]` | 工具白名单，Claude 可无需用户审批直接调用 |
| `ask` | `string[]` | 需要用户审批的工具列表，每次调用前都会弹窗确认 |
| `deny` | `string[]` | 工具黑名单，Claude 完全无法使用的工具 |

#### `allowManagedPermissionRulesOnly`

```json
{ "allowManagedPermissionRulesOnly": true }
```

当设为 `true` 时，**用户级和项目级的权限规则被完全忽略**，仅使用企业托管设置（managed-settings.json）中定义的权限规则。这是防止用户绕过组织策略的关键机制。

#### `allowManagedHooksOnly`

```json
{ "allowManagedHooksOnly": true }
```

当设为 `true` 时，**禁止用户/项目定义自定义 Hooks**，只允许企业管理员通过托管设置配置的 Hooks 生效。防止用户通过恶意 Hook 绕过安全检查。

#### `strictKnownMarketplaces`

```json
{ "strictKnownMarketplaces": [] }
```

Plugin 市场白名单。当设为空数组 `[]` 时，**完全屏蔽所有第三方 Plugin 市场**（仅允许官方内置插件）。当设为包含条目的数组时，只允许白名单内的市场源。

从 CHANGELOG 可知该字段支持两种匹配方式：
- `hostPattern`：按主机名正则匹配市场源域名
- `pathPattern`：按路径正则匹配文件/目录市场源（v2.1.68+ 新增）

### 2.3 `sandbox` 对象

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": false,
    "allowUnsandboxedCommands": false,
    "excludedCommands": [],
    "network": {
      "allowUnixSockets": [],
      "allowAllUnixSockets": false,
      "allowLocalBinding": false,
      "allowedDomains": [],
      "httpProxyPort": null,
      "socksProxyPort": null
    },
    "enableWeakerNestedSandbox": false
  }
}
```

| 字段 | 含义 |
|------|------|
| `enabled` | 是否启用 Bash 沙箱（仅对 Bash 工具生效，不影响 Read/Write/MCP） |
| `autoAllowBashIfSandboxed` | 沙箱模式下是否自动授权 Bash 命令（无需用户审批）|
| `allowUnsandboxedCommands` | 是否允许指定命令跳过沙箱运行 |
| `excludedCommands` | 从沙箱排除的命令列表 |
| `network.allowedDomains` | Bash 沙箱内允许访问的网络域名 |
| `network.allowLocalBinding` | 是否允许沙箱内进程绑定本地端口 |
| `network.allowAllUnixSockets` | 是否允许所有 Unix socket 通信 |
| `enableWeakerNestedSandbox` | 是否对子进程启用较弱的嵌套沙箱 |

> **重要提示**（来自 README）：`sandbox` 属性仅对 `Bash` 工具生效，不影响 `Read`、`Write`、`WebSearch`、`WebFetch`、MCP 服务器、Hooks 或内部命令。

---

## 3. .local.md 插件设置文件

### 3.1 命名约定与加载机制

Hookify 插件通过扫描 `.claude/hookify.*.local.md` 模式的文件来加载用户自定义规则：

```
.claude/
├── hookify.warn-console-log.local.md       ← console.log 警告规则
├── hookify.block-dangerous-rm.local.md     ← 危险命令阻断规则
├── hookify.require-tests-run.local.md      ← 测试强制执行规则
└── hookify.warn-sensitive-files.local.md   ← 敏感文件警告规则
```

**命名格式**：`.claude/hookify.<rule-name>.local.md`

加载代码（`config_loader.py`）：
```python
pattern = os.path.join('.claude', 'hookify.*.local.md')
files = glob.glob(pattern)
```

### 3.2 YAML Frontmatter 结构

每个 `.local.md` 文件由两部分组成：YAML frontmatter（规则配置）和 Markdown 正文（用户提示消息）。

**通用格式模板**：

```markdown
---
name: <规则唯一标识符>
enabled: true | false
event: bash | file | stop | all
pattern: <简单正则模式（legacy）>
action: warn | block
tool_matcher: <工具名 | 工具1|工具2 | *>
conditions:
  - field: command | new_text | file_path | transcript | user_prompt
    operator: regex_match | contains | equals | not_contains | starts_with | ends_with
    pattern: <匹配模式>
---

# Markdown 格式的用户提示消息

消息内容支持完整的 Markdown 语法，会作为 systemMessage 注入 Claude 的上下文。
```

### 3.3 四种 .local.md 规则设计对比

| 维度 | `console-log-warning` | `dangerous-rm` | `require-tests-stop` | `sensitive-files-warning` |
|------|----------------------|----------------|---------------------|--------------------------|
| **名称** | `warn-console-log` | `block-dangerous-rm` | `require-tests-run` | `warn-sensitive-files` |
| **启用状态** | `enabled: true` | `enabled: true` | `enabled: false` | `enabled: true` |
| **触发事件** | `file`（文件写入） | `bash`（命令执行） | `stop`（会话结束） | `file`（文件写入） |
| **动作类型** | `warn`（警告继续） | `block`（阻断拒绝） | `block`（阻断结束） | `warn`（警告继续） |
| **匹配风格** | 简单 `pattern`（legacy） | 简单 `pattern`（legacy） | 复杂 `conditions`（新式） | 复杂 `conditions`（新式） |
| **匹配字段** | `new_text`（自动推断） | `command`（自动推断） | `transcript`（对话记录） | `file_path`（文件路径） |
| **正则模式** | `console\.log\(` | `rm\s+-rf` | `npm test\|pytest\|cargo test` | `\.env$\|\.env\.\|credentials\|secrets` |
| **操作符** | `regex_match`（默认） | `regex_match`（默认） | `not_contains`（逻辑取反） | `regex_match` |
| **安全等级** | 低（代码质量） | 高（数据安全） | 中（质量门控） | 高（凭据保护） |

### 3.4 两种条件语法对比

**Legacy 简单模式**（`console-log-warning`、`dangerous-rm`）：

```yaml
---
name: block-dangerous-rm
event: bash
pattern: rm\s+-rf        # 直接正则，event 决定匹配哪个字段
action: block
---
```

`config_loader.py` 自动推断：`bash` → `command` 字段，`file` → `new_text` 字段，其他 → `content` 字段。

**新式 conditions 模式**（`require-tests-stop`、`sensitive-files-warning`）：

```yaml
---
name: require-tests-run
event: stop
action: block
conditions:
  - field: transcript        # 明确指定字段
    operator: not_contains   # 支持更丰富的操作符
    pattern: npm test|pytest|cargo test
---
```

新式模式支持多条件 AND 组合，字段可精确指定，操作符更丰富（6种）。

### 3.5 规则引擎决策流程

```
Hook 触发 (PreToolUse / Stop / ...)
         │
         ▼
   加载匹配 event 的所有 enabled 规则
         │
         ▼
   遍历每条规则，检查所有 conditions（AND 逻辑）
         │
    ┌────┴────┐
    │  匹配？  │
    └────┬────┘
         │ YES
    ┌────┴──────┐
    │  action?  │
    ├───────────┤
    │  block    │──→ 返回 permissionDecision: "deny" 或 decision: "block"
    │  warn     │──→ 累积 systemMessage（允许继续操作）
    └───────────┘
         │
   合并所有 blocking/warning 消息
         │
         ▼
   注入 systemMessage 到 Claude 上下文
```

---

## 4. Bash 沙箱与网络隔离

### 4.1 settings-bash-sandbox.json 详解

沙箱配置专为完全隔离的执行环境设计：

```json
{
  "allowManagedPermissionRulesOnly": true,   // 锁定权限规则，防止用户绕过
  "sandbox": {
    "enabled": true,                          // 强制启用沙箱
    "autoAllowBashIfSandboxed": false,        // 沙箱内仍需每次审批
    "allowUnsandboxedCommands": false,        // 禁止任何命令跳过沙箱
    "excludedCommands": [],                   // 无例外命令
    "network": {
      "allowUnixSockets": [],                 // 不允许任何 Unix socket
      "allowAllUnixSockets": false,           // 全面禁止 Unix socket
      "allowLocalBinding": false,             // 禁止绑定本地端口
      "allowedDomains": [],                   // 不允许任何网络出站
      "httpProxyPort": null,                  // 无 HTTP 代理
      "socksProxyPort": null                  // 无 SOCKS 代理
    },
    "enableWeakerNestedSandbox": false        // 嵌套进程使用相同严格级别
  }
}
```

**与 strict 模式的关键差异**：沙箱模式不包含 `permissions.disableBypassPermissionsMode`，因为沙箱本身在 OS 级别提供了更底层的隔离，`--dangerously-skip-permissions` 在沙箱中也无法逃脱系统级限制。

### 4.2 init-firewall.sh 防火墙规则分析

DevContainer 启动时（`postStartCommand`）执行防火墙初始化，实现严格的网络隔离：

```
初始化流程：
1. 保存 Docker 内部 DNS 规则（127.0.0.11）
2. 清空所有 iptables 规则（干净状态）
3. 恢复 Docker DNS 规则（保证容器内 DNS 解析）
4. 设置基础放行规则：
   ├── OUTPUT UDP:53  → 允许出站 DNS 查询
   ├── INPUT  UDP:53  → 允许 DNS 响应
   ├── OUTPUT TCP:22  → 允许 SSH 出站
   └── lo 接口        → 允许本地回环
5. 动态构建 IP 白名单（ipset hash:net）：
   ├── 从 api.github.com/meta 获取 GitHub IP 段
   │   （web + api + git 三类，经 aggregate 聚合）
   ├── 解析以下域名的 A 记录：
   │   ├── registry.npmjs.org    (npm 包管理)
   │   ├── api.anthropic.com     (Claude API)
   │   ├── sentry.io             (错误上报)
   │   ├── statsig.anthropic.com (功能开关)
   │   ├── statsig.com           (功能开关)
   │   ├── marketplace.visualstudio.com  (VSCode 扩展)
   │   ├── vscode.blob.core.windows.net  (VSCode 资源)
   │   └── update.code.visualstudio.com  (VSCode 更新)
6. 设置默认策略：INPUT DROP / FORWARD DROP / OUTPUT DROP
7. 放行已建立连接的入/出站流量（ESTABLISHED,RELATED）
8. 放行匹配白名单 IP 的出站流量
9. REJECT 所有其他出站（返回 icmp-admin-prohibited 快速失败）
10. 验证：确认 example.com 不可达，api.github.com 可达
```

**安全亮点**：
- 使用 `ipset hash:net` 支持 CIDR 网段匹配，效率高于单 IP 比对
- GitHub IP 范围动态获取（容器启动时最新），而非硬编码
- 启用严格的 bash 模式（`set -euo pipefail`），任何步骤失败立即退出
- 对获取的所有 IP/CIDR 进行格式校验，防止注入攻击
- 防火墙配置完成后执行双向验证（禁止项 + 允许项）

### 4.3 devcontainer.json 容器级权限控制

```json
{
  "runArgs": [
    "--cap-add=NET_ADMIN",   // 允许修改 iptables（用于 init-firewall.sh）
    "--cap-add=NET_RAW"      // 允许原始套接字（用于网络诊断）
  ],
  "remoteUser": "node",      // 以非 root 用户运行
  "mounts": [
    // Claude 配置独立 volume（不共享到宿主机）
    "source=claude-code-config-${devcontainerId},target=/home/node/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/node/.claude"  // 显式指定配置目录
  },
  "postStartCommand": "sudo /usr/local/bin/init-firewall.sh"  // 以 root 运行防火墙初始化
}
```

**设计要点**：
- 容器以 `node` 非特权用户运行，但防火墙初始化需要 `sudo`（通过 `NET_ADMIN` 能力授权）
- Claude 配置目录挂载为独立 Docker volume（`claude-code-config-*`），防止配置跨会话泄漏到宿主机文件系统
- Bash 历史记录也使用独立 volume（`claude-code-bashhistory-*`），与项目代码隔离

---

## 5. Plugin Marketplace 信任机制

### 5.1 marketplace.json 结构

`.claude-plugin/marketplace.json` 定义了本地 Plugin 市场，作为官方捆绑插件的分发清单：

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "version": "1.0.0",
  "owner": { "name": "Anthropic", "email": "support@anthropic.com" },
  "plugins": [...]
}
```

**已注册的 13 个官方插件**（按类别）：

| 类别 | 插件 | 描述 |
|------|------|------|
| development | `agent-sdk-dev` | Claude Agent SDK 开发套件 |
| development | `claude-opus-4-5-migration` | Opus 版本迁移工具 |
| development | `feature-dev` | 功能开发完整工作流 |
| development | `frontend-design` | 高质量前端界面生成 |
| development | `plugin-dev` | Claude Code 插件开发工具集 |
| development | `ralph-wiggum` | 自引用迭代开发循环 |
| productivity | `code-review` | 多 Agent PR 代码审查 |
| productivity | `commit-commands` | Git commit 工作流命令 |
| productivity | `hookify` | 自定义 Hooks 规则引擎 |
| productivity | `pr-review-toolkit` | PR 审查专项工具集 |
| learning | `explanatory-output-style` | 教育性输出风格（已弃用功能模拟） |
| learning | `learning-output-style` | 交互式学习模式 |
| security | `security-guidance` | 安全问题警告 Hook |

### 5.2 信任链设计

```
┌─────────────────────────────────────────────────┐
│              Plugin 信任链                        │
├─────────────────────────────────────────────────┤
│  Level 1: 内置插件（硬编码）                      │
│  └── 代码编译进 Claude Code 本体，最高信任        │
├─────────────────────────────────────────────────┤
│  Level 2: 官方 Marketplace（.claude-plugin/）    │
│  └── 由 marketplace.json schema 验证              │
│  └── strictKnownMarketplaces 白名单控制           │
├─────────────────────────────────────────────────┤
│  Level 3: 用户添加的第三方 Marketplace            │
│  └── 需要用户显式确认信任                         │
│  └── 可被 strictKnownMarketplaces: [] 完全屏蔽   │
│  └── pluginTrustMessage 可附加组织安全提示        │
├─────────────────────────────────────────────────┤
│  Level 4: 本地文件系统 Marketplace               │
│  └── pathPattern 正则控制                         │
│  └── 仅企业设置层可配置                           │
└─────────────────────────────────────────────────┘
```

---

## 6. 权限设计哲学

### 6.1 三种安全预设代表的权限哲学

**宽松模式（Lax）— "信任但验证"**

哲学核心：相信用户的判断，但防止最危险的系统级漏洞（绕过权限 + 不受控插件）。适合已有外部安全控制（如 SOC2 合规流程、代码审查）的成熟工程团队。用户保留完全的工具使用权和自定义能力。

**严格模式（Strict）— "默认拒绝，显式授权"**

哲学核心：所有高风险操作需要明确的人工审批，敏感工具直接禁用。适合处理医疗、金融等高监管行业数据的团队，或需要完整审计日志的合规场景。每次 Bash 执行都留下批准记录。

**沙箱模式（Bash Sandbox）— "零信任执行环境"**

哲学核心：在 OS 级别隔离代码执行，即使 AI 产生恶意命令也无法造成超出沙箱的破坏。适合自动化 CI/CD 流水线、测试环境，或对 AI 生成代码完全不信任的场景。安全由技术手段保证，不依赖人工审批。

### 6.2 分层防御的设计意图

```
"防御纵深" (Defense in Depth) 原则：
                            
宿主机环境 ←── 防火墙层（init-firewall.sh）
    │                ↓ 只允许白名单出站
容器层      ←── DevContainer 隔离
    │                ↓ NET_ADMIN + 非 root 用户
托管设置层  ←── managed-settings.json
    │                ↓ 企业策略锁定
应用设置层  ←── settings.json
    │                ↓ 工具权限/沙箱配置
Hook 层     ←── hookify .local.md rules
    │                ↓ 实时行为拦截
Claude AI   ←── 模型本身的安全训练
```

每层可以独立提供保护，也可以组合使用。即使某一层被绕过，其他层仍能提供防护。

### 6.3 `disableBypassPermissionsMode` 的特殊意义

这个字段体现了一个重要的安全原则：**即使是安全机制本身，也要防止被绕过**。

`--dangerously-skip-permissions` 本是开发者的便利工具（如 CI/CD 自动化），但在企业环境中它可能被滥用来规避所有安全控制。`disableBypassPermissionsMode: "disable"` 的存在说明：

1. Claude Code 意识到自己的安全机制可能被自身工具绕过
2. 在不可信环境中，必须在系统层面禁用这个"逃生门"
3. 这个设置只有在企业托管设置层才真正有效（用户无法在自己的设置文件中锁定自己）

---

## 7. 关键发现总结

### 7.1 架构层面

1. **四层权限纵深**：从 Docker 防火墙（OS 层）→ 企业托管设置（管理层）→ 应用设置（配置层）→ Hook 规则（运行时层），形成完整的安全防护链。

2. **分离关注点**：`settings.json` 负责"什么可以做"（工具权限），hookify `.local.md` 负责"如何监控和阻断"（行为模式），两者功能互补但机制独立。

3. **企业设置的单向性**：`allowManagedPermissionRulesOnly` 和 `allowManagedHooksOnly` 只能由企业管理员设置，用户无法自行锁定自己——这是"管理者控制用户"而非"用户自我限制"的设计。

### 7.2 功能层面

4. **沙箱的局限性已明确文档化**：沙箱只保护 Bash 工具，不保护 Read/Write/WebFetch/MCP 等其他工具。这避免了"虚假的安全感"——使用沙箱模式仍需配合其他权限控制。

5. **hookify 的两种语法**：简单 `pattern`（legacy）和复杂 `conditions`（新式）并存。新式支持多字段、多操作符的复合条件，特别适合 Stop 事件（检查整个对话历史）。

6. **防火墙的动态 IP 获取**：GitHub IP 范围在容器启动时实时获取而非硬编码，这解决了 GitHub IP 段频繁变更的运维痛点，但也意味着容器启动需要临时网络访问。

### 7.3 安全哲学层面

7. **`strictKnownMarketplaces: []` 的双重含义**：空数组既是"白名单为空"（拒绝所有第三方市场），也是"明确声明了安全意图"——相比不配置该字段，空数组的意图更清晰。

8. **warn vs block 的精心设计**：hookify 的两种 action 反映了"有用性 vs 安全性"的权衡。`warn` 保持工作流畅通但提供可见的提示（console.log、敏感文件），`block` 则对真正危险的操作（`rm -rf`、未运行测试就结束）踩下刹车。

9. **`require-tests-stop` 默认禁用的智慧**：这条规则默认 `enabled: false`，说明设计者意识到"强制测试"在某些场景（探索性编程、原型开发）会严重影响效率，宁愿让用户主动选择开启，而非默认强制。

10. **防火墙验证的"双向测试"**：`init-firewall.sh` 在完成配置后同时验证"example.com 不可达"和"api.github.com 可达"，这种阴性+阳性双向验证避免了"验证逻辑本身也出错"的情况。
