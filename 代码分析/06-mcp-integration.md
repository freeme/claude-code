# 06 - MCP（Model Context Protocol）集成深度分析

> **分析范围**：`/Volumes/Data/agent-frameworks/claude-code/plugins/plugin-dev/skills/mcp-integration/`  
> **涵盖文件**：SKILL.md、examples/（三个配置示例）、references/（server-types.md、authentication.md、tool-usage.md）  
> **生成日期**：2026-04-08

---

## 目录

1. [MCP 概述与定位](#1-mcp-概述与定位)
2. [.mcp.json 配置格式详解](#2-mcpjson-配置格式详解)
3. [MCP Server 类型深度对比](#3-mcp-server-类型深度对比)
4. [MCP 在 Plugin 架构中的位置](#4-mcp-在-plugin-架构中的位置)
5. [MCP 工具命名与暴露机制](#5-mcp-工具命名与暴露机制)
6. [认证方案全解析](#6-认证方案全解析)
7. [集成模式与使用范式](#7-集成模式与使用范式)
8. [生命周期管理](#8-生命周期管理)
9. [性能优化与调试](#9-性能优化与调试)
10. [关键发现总结](#10-关键发现总结)

---

## 1. MCP 概述与定位

**Model Context Protocol（MCP）** 是 Claude Code Plugin 体系中连接外部世界的核心机制。它允许插件将外部服务（数据库、API、文件系统、云平台等）的能力以结构化工具的形式暴露给 Claude，从而大幅扩展其能力边界。

### 核心价值

| 能力维度 | 说明 |
|---------|------|
| **外部服务连接** | 数据库、REST API、文件系统、云平台等 |
| **批量工具提供** | 单个 MCP Server 可以一次性提供 10+ 相关工具 |
| **认证流程处理** | 内置 OAuth 2.0 及 Token 认证管理 |
| **自动随插件分发** | MCP Server 可与插件捆绑，自动配置和启动 |

### MCP 的两种配置方式

#### 方式一：独立 .mcp.json（推荐）

在插件根目录创建 `.mcp.json` 文件（推荐用于多服务器或复杂场景）：

```json
{
  "database-tools": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
    "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
    "env": {
      "DB_URL": "${DB_URL}"
    }
  }
}
```

**适用场景**：多个 Server、关注点分离、便于维护。

#### 方式二：内联在 plugin.json 中

在 `plugin.json` 的 `mcpServers` 字段中直接定义：

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "mcpServers": {
    "plugin-api": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/api-server",
      "args": ["--port", "8080"]
    }
  }
}
```

**适用场景**：单个 Server、配置简单、单文件管理。

---

## 2. .mcp.json 配置格式详解

### 2.1 三种配置示例字段结构对比

以下来自仓库中 `examples/` 目录的真实配置文件：

#### stdio-server.json（本地进程）

```json
{
  "_comment": "Example stdio MCP server configuration for local file system access",
  "filesystem": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "${CLAUDE_PROJECT_DIR}"],
    "env": {
      "LOG_LEVEL": "info"
    }
  },
  "database": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server.js",
    "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/db.json"],
    "env": {
      "DATABASE_URL": "${DATABASE_URL}",
      "DB_POOL_SIZE": "10"
    }
  },
  "custom-tools": {
    "command": "python",
    "args": ["-m", "my_mcp_server", "--port", "8080"],
    "env": {
      "API_KEY": "${CUSTOM_API_KEY}",
      "DEBUG": "false"
    }
  }
}
```

#### sse-server.json（Server-Sent Events）

```json
{
  "_comment": "Example SSE MCP server configuration for hosted cloud services",
  "asana": {
    "type": "sse",
    "url": "https://mcp.asana.com/sse"
  },
  "github": {
    "type": "sse",
    "url": "https://mcp.github.com/sse"
  },
  "custom-service": {
    "type": "sse",
    "url": "https://mcp.example.com/sse",
    "headers": {
      "X-API-Version": "v1",
      "X-Client-ID": "${CLIENT_ID}"
    }
  }
}
```

#### http-server.json（REST API）

```json
{
  "_comment": "Example HTTP MCP server configuration for REST APIs",
  "rest-api": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}",
      "Content-Type": "application/json",
      "X-API-Version": "2024-01-01"
    }
  },
  "internal-service": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}",
      "X-Service-Name": "claude-plugin"
    }
  }
}
```

### 2.2 核心字段详解

| 字段 | 类型 | 适用类型 | 说明 |
|------|------|---------|------|
| `command` | string | stdio | 要执行的命令（如 `npx`、`python`、可执行文件路径） |
| `args` | string[] | stdio | 传递给命令的参数列表 |
| `env` | object | stdio | 传递给子进程的环境变量键值对 |
| `type` | string | SSE/HTTP/WS | Server 类型标识：`"sse"`、`"http"`、`"ws"` |
| `url` | string | SSE/HTTP/WS | Server 的连接地址（必须 HTTPS/WSS） |
| `headers` | object | SSE/HTTP/WS | HTTP 请求头，用于认证或版本控制 |
| `headersHelper` | string | SSE/HTTP/WS | 动态生成请求头的脚本路径（高级用法） |

> **注意**：stdio 类型不需要 `type` 字段（默认即为 stdio）；其他类型必须显式指定 `type`。

### 2.3 环境变量展开机制

MCP 配置支持两类环境变量替换：

**`${CLAUDE_PLUGIN_ROOT}`**：插件安装目录的绝对路径（系统注入，用于可移植路径引用）：

```json
{
  "command": "${CLAUDE_PLUGIN_ROOT}/servers/my-server",
  "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/settings.json"]
}
```

**用户 Shell 环境变量**：从用户 Shell 环境中读取（用于密钥、连接串等敏感信息）：

```json
{
  "env": {
    "API_KEY": "${MY_API_KEY}",
    "DATABASE_URL": "${DB_URL}"
  }
}
```

### 2.4 项目级 .mcp.json 与全局 MCP 配置的关系

- **项目级 `.mcp.json`**：位于插件根目录，随插件一起分发，作用域仅限于该插件。
- **`plugin.json` 中的 `mcpServers`**：等价于 `.mcp.json`，可通过字符串路径引用外部文件（`"mcpServers": "./.mcp.json"`）或直接内联对象。
- **全局 MCP 配置**：Claude Code 本身也维护用户级别的全局 MCP 配置；插件配置是在其基础上叠加注册的，不会互相覆盖。
- **自动发现**：Claude Code 在插件加载时自动扫描 `.mcp.json` 或 `plugin.json` 中的 `mcpServers` 字段，无需手动注册。

---

## 3. MCP Server 类型深度对比

### 3.1 四种类型全面对比表

| 特性 | stdio | SSE | HTTP | WebSocket |
|------|-------|-----|------|-----------|
| **传输层** | 进程 stdin/stdout | HTTP + Server-Sent Events | REST HTTP | WebSocket |
| **通信方向** | 双向 | 服务端→客户端推送 | 请求/响应（单向） | 双向 |
| **状态模型** | 有状态 | 有状态 | 无状态 | 有状态 |
| **认证方式** | 环境变量 | OAuth / 自定义 Headers | Bearer Token / API Key | Bearer Token / 自定义 |
| **延迟** | 最低（本地进程） | 中等 | 中等 | 低 |
| **配置复杂度** | 简单 | 中等 | 简单 | 中等 |
| **断线重连** | 进程重启 | 自动重连 | 不适用（无状态） | 自动重连 |
| **典型场景** | 本地工具、自定义服务器 | 云服务、OAuth 官方服务 | REST API 后端 | 实时流数据 |
| **是否需要服务器** | 否（本地运行） | 是（远程） | 是（远程） | 是（远程） |
| **进程管理** | Claude Code 负责启停 | 外部服务自管理 | 外部服务自管理 | 外部服务自管理 |

### 3.2 各类型详解

#### stdio（本地进程）

Claude Code 将 MCP Server 作为子进程启动，通过 stdin/stdout 以 JSON-RPC 协议通信。

**进程生命周期**：
1. Claude Code 启动时用 `command` + `args` 创建子进程
2. 通过 stdin/stdout 进行 JSON-RPC 消息交换
3. 整个 Claude Code 会话期间进程持续运行
4. Claude Code 退出时子进程随之终止

**典型使用场景**：
- 文件系统访问（`@modelcontextprotocol/server-filesystem`）
- 本地数据库连接
- 自定义 Node.js / Python MCP Server
- NPM 发布的 MCP Server 包

**注意事项**：
- 子进程的 stdout 被 MCP 协议占用，日志必须写入 stderr
- Python Server 需设置 `PYTHONUNBUFFERED=1` 避免缓冲问题
- 路径必须使用 `${CLAUDE_PLUGIN_ROOT}` 保证可移植性

#### SSE（Server-Sent Events）

通过 HTTP 长连接连接到已托管的 MCP Server，Server 端以 SSE 格式推送事件。最适合官方云服务集成。

**连接生命周期**：
1. Claude Code 向 URL 发起 HTTP 连接
2. MCP 协议握手与协商
3. Server 通过 SSE 持续推送事件
4. 客户端通过 HTTP POST 发起工具调用
5. 断线时自动重连

**典型使用场景**：
- 官方托管 MCP Server（Asana: `https://mcp.asana.com/sse`、GitHub: `https://mcp.github.com/sse`）
- 需要 OAuth 认证的云服务
- 无需本地安装的外部平台集成

#### HTTP（REST API）

通过标准 HTTP 请求连接 RESTful MCP Server，每次工具调用是独立的 HTTP 请求。

**请求/响应流程**：
1. 工具发现：GET 请求获取可用工具列表
2. 工具调用：POST 请求携带工具名称和参数
3. 接收响应：JSON 格式的结果或错误
4. 无状态：每次请求相互独立

**典型使用场景**：
- 内部微服务 API
- Serverless Functions
- 需要 Token 认证的 REST API 后端

#### WebSocket（实时双向通信）

通过 WebSocket 与 MCP Server 建立持久双向连接，适合低延迟和实时数据场景。

**连接生命周期**：
1. WebSocket 升级握手
2. 建立持久双向信道
3. JSON-RPC 消息通过 WebSocket 传输
4. 心跳保活（ping/pong）
5. 断线时自动重连

**典型使用场景**：
- 实时数据流（行情、日志流）
- 协同编辑类应用
- 低延迟工具调用
- 服务端主动推送通知

### 3.3 选型决策树

```
需要连接的服务是本地的吗？
    ├── 是 → 使用 stdio
    └── 否
         ├── 需要 OAuth 认证或官方云服务？
         │    ├── 是 → 使用 SSE
         │    └── 否
         │         ├── 需要实时双向通信 / 低延迟？
         │         │    ├── 是 → 使用 WebSocket
         │         │    └── 否 → 使用 HTTP
```

---

## 4. MCP 在 Plugin 架构中的位置

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         Claude Code 运行时                               ║
║                                                                          ║
║  ┌────────────────────────────────────────────────────────────────────┐  ║
║  │                         Plugin System                              │  ║
║  │                                                                    │  ║
║  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │  ║
║  │  │   Commands   │  │    Agents    │  │    Skills    │             │  ║
║  │  │  (*.md 文件)  │  │  (*.md 文件)  │  │  (SKILL.md)  │             │  ║
║  │  └──────┬───────┘  └──────┬───────┘  └──────────────┘             │  ║
║  │         │                 │                                        │  ║
║  │         │  allowed-tools  │  tool references                      │  ║
║  │         ▼                 ▼                                        │  ║
║  │  ┌─────────────────────────────────────────────────────────────┐  │  ║
║  │  │              Tool Registry (工具注册表)                       │  │  ║
║  │  │                                                             │  │  ║
║  │  │  ┌─────────────────┐    ┌──────────────────────────────┐   │  │  ║
║  │  │  │  Built-in Tools │    │       MCP Tools              │   │  │  ║
║  │  │  │  - Read         │    │  mcp__plugin_<name>_<srv>__  │   │  │  ║
║  │  │  │  - Write        │    │  ┌────────────────────────┐  │   │  │  ║
║  │  │  │  - Bash         │    │  │ asana_create_task      │  │   │  │  ║
║  │  │  │  - ...          │    │  │ k8s_list_pods          │  │   │  │  ║
║  │  │  └─────────────────┘    │  │ db_query               │  │   │  │  ║
║  │  │                         │  └────────────────────────┘  │   │  │  ║
║  │  │                         └──────────────────────────────┘   │  │  ║
║  │  └─────────────────────────────────────────────────────────────┘  │  ║
║  │                                                                    │  ║
║  │  ┌──────────────────────────────────────────────────────────────┐ │  ║
║  │  │              MCP Configuration Layer                         │ │  ║
║  │  │                                                              │ │  ║
║  │  │  plugin.json ──► mcpServers ──► .mcp.json                   │ │  ║
║  │  └──────────────────────────────┬───────────────────────────────┘ │  ║
║  └────────────────────────────────│────────────────────────────────── ┘  ║
║                                   │                                       ║
║         ┌─────────────────────────▼─────────────────────────────┐        ║
║         │                MCP Server 层                           │        ║
║         │                                                        │        ║
║         │  ┌───────────┐  ┌───────────┐  ┌─────────┐  ┌──────┐ │        ║
║         │  │   stdio   │  │    SSE    │  │  HTTP   │  │  WS  │ │        ║
║         │  │(子进程)   │  │ (长连接)  │  │(无状态) │  │(双向)│ │        ║
║         │  └─────┬─────┘  └─────┬─────┘  └────┬────┘  └──┬───┘ │        ║
║         └────────│──────────────│──────────────│──────────│──────┘        ║
║                  │              │              │           │               ║
╚══════════════════│══════════════│══════════════│═══════════│═══════════════╝
                   │              │              │           │
         ┌─────────▼──┐  ┌───────▼──────┐  ┌───▼──────┐  ┌▼──────────────┐
         │ 本地工具/  │  │ Asana/GitHub │  │ REST API │  │ 实时数据流   │
         │ 自定义脚本 │  │ 官方 MCP 服务 │  │ 内部服务 │  │ WebSocket 服务│
         └────────────┘  └──────────────┘  └──────────┘  └───────────────┘
```

### Plugin 组件与 MCP 的协作关系

- **Commands（命令）**：在 frontmatter 的 `allowed-tools` 字段中预声明需要使用的 MCP 工具，触发时受权限控制
- **Agents（代理）**：可以自主决定调用哪些 MCP 工具，无需预先声明，权限更宽松
- **Skills（技能）**：提供上下文知识和操作指导，间接影响 MCP 工具的使用方式
- **Hooks（钩子）**：在 MCP 工具调用前后触发，可以拦截、验证或增强调用行为

---

## 5. MCP 工具命名与暴露机制

### 5.1 工具命名规则

MCP Server 提供的工具在注册后会自动加上前缀，格式如下：

```
mcp__plugin_<plugin-name>_<server-name>__<tool-name>
```

**命名示例**：

| 插件名 | Server 名 | 原始工具名 | 完整工具名 |
|-------|-----------|----------|-----------|
| `asana` | `asana` | `create_task` | `mcp__plugin_asana_asana__asana_create_task` |
| `asana` | `asana` | `search_tasks` | `mcp__plugin_asana_asana__asana_search_tasks` |
| `myplug` | `database` | `query` | `mcp__plugin_myplug_database__query` |
| `myplug` | `database` | `list_tables` | `mcp__plugin_myplug_database__list_tables` |

### 5.2 在 Commands 中预声明工具

```markdown
---
description: Create a new Asana task
allowed-tools: [
  "mcp__plugin_asana_asana__asana_create_task",
  "mcp__plugin_asana_asana__asana_search_tasks"
]
---
```

**通配符**（谨慎使用）：

```markdown
---
allowed-tools: ["mcp__plugin_asana_asana__*"]
---
```

> **安全原则**：始终预声明特定工具，而非使用通配符，以最小化权限暴露面。

### 5.3 在 Agents 中使用工具

Agent 拥有比 Command 更宽泛的工具访问权限，可以在 Agent 描述中以自然语言引用 MCP 工具，Claude 会自动识别和调用：

```markdown
---
name: asana-status-updater
description: Use when user asks to update Asana status or generate project report
---

## Process
1. Query tasks: Use mcp__plugin_asana_asana__asana_search_tasks
2. Analyze progress and calculate completion rates
3. Use mcp__plugin_asana_asana__asana_create_comment to post report
```

### 5.4 工具发现

使用 `/mcp` 命令可以查看所有已注册的 MCP Server 及其提供的工具：

```bash
/mcp
```

该命令显示：
- 所有可用 MCP Server 列表
- 每个 Server 提供的工具
- 工具的 Schema（参数定义）
- 可在配置中直接使用的完整工具名

---

## 6. 认证方案全解析

### 6.1 四种认证方式对比

| 认证方式 | 适用 Server 类型 | 配置复杂度 | 安全级别 | 用户体验 |
|---------|----------------|---------|---------|---------|
| **OAuth 2.0（自动）** | SSE、HTTP | 无需额外配置 | 高（短期 Token + 自动刷新） | 最佳（首次浏览器授权后自动） |
| **Bearer Token** | HTTP、WebSocket | 低 | 中（取决于 Token 管理） | 良好 |
| **API Key** | HTTP、WebSocket | 低 | 中 | 良好 |
| **环境变量注入** | stdio | 低 | 依赖用户系统安全 | 需要手动配置 |

### 6.2 OAuth 2.0（自动处理）

针对支持 OAuth 的 SSE 或 HTTP Server，Claude Code **全程自动处理** OAuth 2.0 流程：

```json
{
  "asana": {
    "type": "sse",
    "url": "https://mcp.asana.com/sse"
  }
}
```

**完整 OAuth 流程**：
1. 用户首次尝试调用 MCP 工具
2. Claude Code 检测到需要认证
3. 自动打开浏览器进行 OAuth 授权
4. 用户在浏览器中授权
5. Token 由 Claude Code **加密存储**（插件无法直接访问）
6. 后续请求自动携带 Token，过期后自动刷新

**已知支持 OAuth 的服务**：
- Asana: `https://mcp.asana.com/sse`
- GitHub MCP（当可用时）
- Google 服务 MCP（当可用时）

### 6.3 Bearer Token 认证

```json
{
  "api": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}"
    }
  }
}
```

用户需在 Shell 中设置环境变量：

```bash
export API_TOKEN="your-secret-token-here"
```

### 6.4 API Key 认证

```json
{
  "api": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "X-API-Key": "${API_KEY}",
      "X-API-Secret": "${API_SECRET}"
    }
  }
}
```

### 6.5 stdio 环境变量注入

```json
{
  "database": {
    "command": "python",
    "args": ["-m", "mcp_server_db"],
    "env": {
      "DATABASE_URL": "${DATABASE_URL}",
      "DB_USER": "${DB_USER}",
      "DB_PASSWORD": "${DB_PASSWORD}"
    }
  }
}
```

### 6.6 动态 Headers（高级）

对于短期有效 Token 或需要 HMAC 签名的场景，可以使用 `headersHelper` 脚本动态生成请求头：

```json
{
  "api": {
    "type": "sse",
    "url": "https://api.example.com",
    "headersHelper": "${CLAUDE_PLUGIN_ROOT}/scripts/get-headers.sh"
  }
}
```

**脚本示例**（HMAC 签名）：

```bash
#!/bin/bash
TIMESTAMP=$(date -Iseconds)
SIGNATURE=$(echo -n "$TIMESTAMP" | openssl dgst -sha256 -hmac "$SECRET_KEY" | cut -d' ' -f2)

cat <<EOF
{
  "X-Timestamp": "$TIMESTAMP",
  "X-Signature": "$SIGNATURE",
  "X-API-Key": "$API_KEY"
}
EOF
```

### 6.7 高级认证：mTLS 与 JWT

**mTLS**：MCP 配置不直接支持，推荐通过 stdio 包装层处理：

```json
{
  "secure-api": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/mtls-wrapper",
    "args": ["--cert", "${CLIENT_CERT}", "--key", "${CLIENT_KEY}"]
  }
}
```

**JWT Token**：通过 `headersHelper` 脚本动态生成，适合需要时间戳签名的 JWT。

### 6.8 安全最佳实践

| 建议 | 禁忌 |
|------|------|
| ✅ 使用环境变量存储 Token | ❌ 在配置文件中硬编码 Token |
| ✅ 优先使用 OAuth（用户体验最佳） | ❌ 将 Token 提交到 Git |
| ✅ 在 README 中说明所需环境变量 | ❌ 在文档示例中暴露真实 Token |
| ✅ 只使用 HTTPS / WSS | ❌ 使用 HTTP / WS 明文传输 |
| ✅ 定期轮换 Token | ❌ 跳过 SSL 证书验证 |

---

## 7. 集成模式与使用范式

### 7.1 模式一：简单工具包装器（Command 驱动）

适合需要在 MCP 调用前进行用户交互或参数验证的场景：

```markdown
---
description: Create a new task in project management tool
allowed-tools: ["mcp__plugin_name_server__create_item"]
---

Steps:
1. Gather item details from user (title, description, priority)
2. Validate required fields are provided
3. Use mcp__plugin_name_server__create_item with validated data
4. Show confirmation with item link
```

### 7.2 模式二：自主代理（Agent 驱动）

适合多步骤、无需用户干预的自动化工作流：

```markdown
---
name: data-analyzer
description: Autonomously analyzes data and generates insights
---

Analysis Process:
1. Query data via mcp__plugin_db_server__query
2. Apply statistical analysis
3. Generate insights report
4. Upload report via mcp__plugin_storage__upload
```

### 7.3 模式三：多 Server 协同（企业级场景）

来自仓库 `advanced-plugin.md` 的真实案例——企业 DevOps 插件的 `.mcp.json`：

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/kubernetes-mcp/index.js"],
      "env": {
        "KUBECONFIG": "${KUBECONFIG}",
        "K8S_NAMESPACE": "${K8S_NAMESPACE:-default}"
      }
    },
    "terraform": {
      "command": "python",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/terraform-mcp/main.py"],
      "env": {
        "TF_STATE_BUCKET": "${TF_STATE_BUCKET}",
        "AWS_REGION": "${AWS_REGION}"
      }
    },
    "github-actions": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/github-actions-mcp/server.js"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "GITHUB_ORG": "${GITHUB_ORG}"
      }
    }
  }
}
```

此插件通过三个 stdio MCP Server 同时集成了 Kubernetes 集群操作、Terraform 基础设施管理和 GitHub Actions CI/CD 流水线。

### 7.4 模式四：跨服务类型混合（生产环境推荐）

同时使用多种 Server 类型满足不同需求：

```json
{
  "local-db": {
    "command": "npx",
    "args": ["-y", "mcp-server-sqlite", "./data.db"]
  },
  "cloud-api": {
    "type": "sse",
    "url": "https://mcp.example.com/sse"
  },
  "internal-service": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}"
    }
  }
}
```

---

## 8. 生命周期管理

### 8.1 完整启动流程

```
插件加载
    │
    ▼
解析 MCP 配置（.mcp.json 或 plugin.json 中的 mcpServers）
    │
    ├── stdio 类型 ──► 启动子进程（spawn command + args）
    │
    ├── SSE 类型   ──► 建立 HTTP 长连接到 URL
    │
    ├── HTTP 类型  ──► 准备 HTTP 客户端（首次工具调用时连接）
    │
    └── WS 类型    ──► 建立 WebSocket 连接
              │
              ▼
    工具发现（从 Server 获取工具列表和 Schema）
              │
              ▼
    工具注册（以 mcp__plugin_... 前缀注册到 Tool Registry）
              │
              ▼
    工具在 Commands / Agents 中可用
```

### 8.2 懒加载策略

- MCP Server **不一定在插件启动时立即全部连接**
- 首次工具调用触发相应 Server 的连接建立
- 连接池由 Claude Code 自动管理
- 配置变更后需要重启 Claude Code 才能生效

### 8.3 断线处理

| Server 类型 | 断线处理策略 |
|------------|------------|
| stdio | 进程崩溃后可重新生成（process respawn） |
| SSE | 自动重连（reconnection 由协议保证） |
| HTTP | 无状态，直接重试请求 |
| WebSocket | 自动重连 + 断线期间消息缓冲 |

---

## 9. 性能优化与调试

### 9.1 性能最佳实践

**批量查询**（减少往返次数）：

```markdown
# 推荐：单次查询加过滤条件
tasks = search_tasks(project="X", assignee="me", limit=50)

# 避免：循环逐个查询
for id in task_ids:
    task = get_task(id)
```

**并行工具调用**：

```markdown
# Claude 会自动并行执行相互独立的工具调用
- mcp__plugin_api__get_project
- mcp__plugin_api__get_users
- mcp__plugin_api__get_tags
# 等全部完成后合并结果
```

**结果复用**：同一会话中对昂贵操作的结果进行缓存，避免重复调用。

### 9.2 调试方法

**启用 Debug 日志**：

```bash
claude --debug
```

日志中关注：
- MCP Server 连接尝试记录
- 工具发现日志
- OAuth 流程进度
- 工具调用错误详情

**验证 Server 状态**：

```bash
/mcp
```

**独立验证认证**：

```bash
curl -H "Authorization: Bearer $API_TOKEN" \
     https://api.example.com/mcp/health
```

### 9.3 常见问题排查

| 问题现象 | 排查方向 |
|---------|---------|
| Server 不出现在 `/mcp` 列表 | 检查配置 JSON 语法、重启 Claude Code |
| 工具调用返回 401 | Token 是否设置、格式是否正确（`Bearer ` 前缀） |
| 工具调用返回 403 | Token 权限范围是否包含所需操作 |
| stdio 进程无法启动 | 检查 command 路径、可执行权限 |
| SSE 连接超时 | 检查 URL、网络可达性、HTTPS 证书 |
| Python Server 无输出 | 设置 `PYTHONUNBUFFERED=1` |

---

## 10. 关键发现总结

### 10.1 架构设计洞察

1. **MCP 是 Plugin 能力边界的核心扩展点**：内置工具（Read/Write/Bash 等）提供编码层面的能力，MCP 工具提供外部系统集成能力，两者共同构成 Claude Code 的完整工具体系。

2. **双配置路径设计**：`.mcp.json` 独立文件与 `plugin.json` 内联配置的双轨机制，在简单场景和复杂场景之间取得了平衡——简单插件无需额外文件，复杂插件保持清晰的关注点分离。

3. **`${CLAUDE_PLUGIN_ROOT}` 是可移植性的关键**：所有文件路径引用均通过此变量，确保插件在不同用户环境中正确运行，这是分发式插件的必要设计。

4. **工具命名前缀机制防止冲突**：`mcp__plugin_<plugin>_<server>__<tool>` 的三级命名空间彻底避免了多插件间的工具名冲突，是多租户插件系统的重要基础。

### 10.2 认证方案选型建议

- **优先 OAuth**：对于支持 OAuth 的官方服务（Asana、GitHub 等），SSE + OAuth 是对用户最友好的方案，Claude Code 全程托管认证生命周期。
- **其次环境变量**：对于 stdio 类型或需要静态 Token 的场景，环境变量是最安全且对开发者最简单的方案。
- **谨慎使用动态 Headers**：`headersHelper` 脚本虽然灵活，但增加了维护复杂度，仅在确实需要动态签名时使用。

### 10.3 实际应用边界扩展

通过 MCP，Claude Code 可以超越纯代码编辑，成为：

| 扩展方向 | MCP 能力支撑 | 示例 |
|---------|------------|------|
| **DevOps 自动化** | stdio（本地工具）+ SSE（CI/CD 平台） | Kubernetes 部署、Terraform 变更 |
| **项目管理集成** | SSE（官方 OAuth 服务） | Asana 任务同步、GitHub Issue 管理 |
| **数据库操作** | stdio（本地 DB Server） | SQL 查询、Schema 分析 |
| **文件系统访问** | stdio（官方 filesystem server） | 跨目录文件操作 |
| **实时监控** | WebSocket（数据流服务） | 日志流分析、指标监控 |

### 10.4 安全风险与对策

| 风险点 | 缓解措施 |
|-------|---------|
| Token 泄露 | 强制使用环境变量，禁止硬编码 |
| 权限过度授予 | `allowed-tools` 精确声明，禁用通配符 |
| 明文传输 | 强制 HTTPS/WSS，拒绝 HTTP/WS |
| 本地命令注入（stdio） | 固定 command 路径，不接受用户输入作为命令参数 |
| 证书问题 | 不跳过 SSL 验证，使用正规 CA 签发证书 |

### 10.5 与 Plugin 其他组件的协同要点

- **Commands**：通过 `allowed-tools` frontmatter 进行精确权限控制，是 MCP 工具的"受控入口"
- **Agents**：工具访问权限更宽泛，是 MCP 工具的"自主执行者"，适合多步骤工作流
- **Hooks**：可以在 MCP 工具调用前后插入验证逻辑，构建安全防线
- **Skills**：提供 MCP 工具的使用上下文和最佳实践，提升工具调用质量

---

*本文档基于 `plugins/plugin-dev/skills/mcp-integration/` 目录下的 SKILL.md、examples/、references/ 全部文件分析生成，并结合 `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md` 中的企业级真实案例进行了验证和补充。*
