# Claude Code Hooks 系统深度分析

> **分析范围**：`plugins/hookify/`、`plugins/security-guidance/`、`plugins/ralph-wiggum/`、`plugins/explanatory-output-style/`、`plugins/learning-output-style/`、`examples/hooks/`
>
> **生成日期**：2026-04-08
>
> **实验记录**：本文档编写过程中，security-guidance 的 PreToolUse Hook **实际触发并阻断了三次**文件写入操作，分别检测到：Node.js 进程生成 API 引用、JS 动态函数构造器引用、Python 序列化模块关键词。这是 Hook 系统实时工作效果的完整验证。

---

## 目录

1. Hook 事件类型全景
2. Hook 生命周期时序图
3. Hook 输入/输出协议
4. Hookify 插件深度分析
5. hooks.json 配置格式详解
6. 安全 Hook 实现案例
7. SessionStart Hook：上下文注入机制
8. 关键发现总结

---

## 1. Hook 事件类型全景

Claude Code 的 Hooks 系统支持以下 9 种事件类型：

| 事件类型 | 触发时机 | 可阻断 | 主要用途 |
|---------|---------|--------|---------|
| **SessionStart** | 会话启动时 | 否 | 注入上下文、初始化状态 |
| **SessionEnd** | 会话结束时 | 否 | 清理资源、记录日志 |
| **UserPromptSubmit** | 用户提交 Prompt 前 | 是（exit 2） | 提示词校验、敏感词过滤 |
| **PreToolUse** | 工具调用前 | 是（exit 2 / JSON deny） | 危险命令拦截、安全审计 |
| **PostToolUse** | 工具调用后 | 是（exit 2） | 结果验证、日志记录 |
| **Stop** | Agent 准备停止时 | 是（JSON block） | 强制执行检查、实现循环 |
| **SubagentStop** | 子 Agent 停止时 | 否 | 子 Agent 结果聚合 |
| **PreCompact** | 对话压缩前 | 否 | 保留关键信息 |
| **Notification** | 系统通知时 | 否 | 消息分发、状态更新 |

### 各事件在插件中的实际使用情况

| 事件类型 | 使用插件 | 实现语言 |
|---------|---------|---------|
| PreToolUse | hookify、security-guidance | Python |
| PostToolUse | hookify | Python |
| Stop | hookify、ralph-wiggum | Python / Bash |
| UserPromptSubmit | hookify | Python |
| SessionStart | explanatory-output-style、learning-output-style | Bash |

---

## 2. Hook 生命周期时序图

### 完整 Agent 会话生命周期

```
用户启动 Claude Code
         |
         v
+---------------------+
|   SessionStart      |  <- 注入上下文（additionalContext）
|   Hooks 触发        |    explanatory / learning 模式在此注入风格指令
+---------------------+
         |
         v
+---------------------------------------------------+
|              用户提交 Prompt                      |
|                    |                              |
|                    v                              |
|         UserPromptSubmit Hooks 触发               |
|          hookify userpromptsubmit.py              |
|          加载 event='prompt' 的规则               |
|                    |                              |
|         +----------+-----------+                  |
|         | exit 0/1  允许继续  |                  |
|         | exit 2    阻断执行  | <- stderr->Claude |
|         +--------------------+                   |
+---------------------------------------------------+
         |
         v (Claude 决定使用工具)
+-----------------------------------------------------------+
|    PreToolUse Hooks 触发                                  |
|         |                                                 |
|  +------+------------------------------------------+     |
|  | Bash 工具      Edit/Write/MultiEdit 工具   其他  |     |
|  | event='bash'   event='file'                     |     |
|  +------+------------------------------------------+     |
|         |                                                 |
|  +------+--------------------------------------------+   |
|  | exit 0    -> 允许工具执行                         |   |
|  | exit 1    -> stderr 显示给用户（不阻断 Claude）   |   |
|  | exit 2    -> 阻断 + stderr 内容反馈给 Claude      |   |
|  | JSON deny -> hookSpecificOutput.permissionDecision|   |
|  +---------------------------------------------------+   |
+-----------------------------------------------------------+
         | (工具执行完成)
         v
+---------------------+
|    PostToolUse      |  <- hookify posttooluse.py
|    Hooks 触发       |    验证工具结果、记录日志
+---------------------+
         |
         v (Claude 决定停止)
+---------------------------------------------------+
|      Stop Hooks 触发                              |
|      hookify stop.py                              |
|      ralph-wiggum stop-hook.sh                    |
|              |                                    |
|   +----------+----------+                         |
|   | exit 0    允许停止  |                         |
|   | JSON block 阻断停止 | <- reason 字段->新Prompt |
|   +---------------------+                         |
+---------------------------------------------------+
         | (允许停止)
         v
+---------------------+
|    SessionEnd       |  <- 清理资源、写入日志
+---------------------+
```

### PreToolUse 规则匹配内部流程（hookify）

```
stdin JSON
    |
    v
tool_name == 'Bash'                       -> event = 'bash'
tool_name in ['Edit','Write','MultiEdit'] -> event = 'file'
其他工具                                  -> event = None
    |
    v
load_rules(event=event)
    |  glob('.claude/hookify.*.local.md')
    |  解析 YAML frontmatter
    |  按 event 字段过滤（bash / file / stop / all）
    |  只加载 enabled=true 的规则
    v
RuleEngine.evaluate_rules(rules, input_data)
    |
    +-- 对每条 Rule:
    |    +-- tool_matcher 匹配？（Bash / Edit|Write / *）
    |    +-- 所有 conditions 全部匹配？（AND 关系）
    |         +-- regex_match -> compile_regex(lru_cache) -> re.search
    |         +-- contains / not_contains
    |         +-- equals / starts_with / ends_with
    |         +-- 字段提取：command / new_string / file_path / reason / transcript
    |
    +-- blocking_rules 存在？
    |    +-- Stop event      -> {"decision":"block","reason":"..."}
    |    +-- Pre/PostToolUse -> {"hookSpecificOutput":{"permissionDecision":"deny"}}
    |    +-- 其他事件        -> {"systemMessage":"..."}
    |
    +-- warning_rules 存在？（无 blocking 时）
         +-- {"systemMessage": "所有警告消息合并（Markdown 格式）"}
```

---

## 3. Hook 输入/输出协议

### 3.1 stdin 输入 JSON 结构

Claude Code 通过 stdin 向 Hook 进程传入结构化 JSON，不同事件类型的字段有所不同：

#### PreToolUse / PostToolUse 事件（Bash 工具）

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "abc123",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  }
}
```

#### PreToolUse / PostToolUse 事件（文件编辑工具）

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "abc123",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "src/auth/login.ts",
    "old_string": "旧代码内容",
    "new_string": "新代码内容"
  }
}
```

#### MultiEdit 工具

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "MultiEdit",
  "tool_input": {
    "file_path": "src/utils.ts",
    "edits": [
      {"old_string": "...", "new_string": "..."},
      {"old_string": "...", "new_string": "..."}
    ]
  }
}
```

对于 MultiEdit，rule_engine 会将所有 edits 的 `new_string` 拼接后进行整体匹配。

#### Stop 事件

```json
{
  "hook_event_name": "Stop",
  "session_id": "abc123",
  "transcript_path": "/tmp/claude-transcript-abc123.jsonl",
  "reason": "Agent completed task"
}
```

`transcript_path` 指向 JSONL 格式的对话记录文件，每行一个 JSON 对象。ralph-wiggum 直接读取该文件，提取最后一条 assistant 消息，实现自循环。

#### UserPromptSubmit 事件

```json
{
  "hook_event_name": "UserPromptSubmit",
  "session_id": "abc123",
  "user_prompt": "请帮我删除所有的日志文件"
}
```

#### SessionStart 事件

```json
{
  "hook_event_name": "SessionStart",
  "session_id": "abc123"
}
```

### 3.2 stdout 响应格式

#### 通用响应字段

| 字段 | 类型 | 说明 |
|-----|------|------|
| `systemMessage` | string | 以 system message 形式注入给 Claude 的文本 |
| `hookSpecificOutput` | object | 事件专属的结构化输出 |

#### PreToolUse / PostToolUse 专属响应（权限拒绝）

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny"
  },
  "systemMessage": "⚠️ 危险命令被阻断：检测到 rm -rf，请确认路径后重试"
}
```

`permissionDecision` 可选 `"allow"` 或 `"deny"`。

#### Stop 专属响应（阻断并继续循环）

```json
{
  "decision": "block",
  "reason": "继续执行的提示词文本（作为新用户消息重新投递给 Claude）",
  "systemMessage": "🔄 Ralph iteration 3 | 目标：完成所有单元测试"
}
```

`decision: "block"` 阻止 Agent 停止，`reason` 字段内容被作为新 Prompt 发回 Claude，实现自循环。

#### SessionStart 专属响应（上下文注入）

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "你处于 explanatory 模式，在编写代码前后提供教育性说明..."
  }
}
```

`additionalContext` 被动态注入到 Claude 的上下文，效果等同于追加 system prompt。

### 3.3 退出码语义

| 退出码 | 含义 | 用户可见 | Claude 可见 | 是否阻断 |
|-------|------|---------|------------|---------|
| `0` | 成功，允许继续 | 否 | 仅 stdout JSON | 否 |
| `1` | 警告（仅用户可见） | 是（stderr） | 否 | 否 |
| `2` | 阻断（Claude 可见） | 否 | 是（stderr 全文） | **是** |

**关键设计**：
- **exit 1**：用户收到 stderr 提示，但 Claude 继续正常运行，适合显示给人看的警告
- **exit 2**：stderr 内容直接成为 Claude 的 feedback，Claude 会根据反馈调整策略

来自 `examples/hooks/bash_command_validator_example.py` 的实现：

```python
def main():
    input_data = json.load(sys.stdin)
    command = input_data.get("tool_input", {}).get("command", "")

    issues = _validate_command(command)
    if issues:
        for msg in issues:
            print(f"• {msg}", file=sys.stderr)
        # exit 2: 阻断工具调用，将 stderr 反馈给 Claude
        sys.exit(2)
```

---

## 4. Hookify 插件深度分析

Hookify 是一个通用声明式规则引擎插件，用户通过编写 `.claude/hookify.*.local.md` 文件定义规则，无需写 Python 代码。

### 4.1 整体架构

```
hookify/
├── hooks/
│   ├── pretooluse.py       <- PreToolUse 入口（事件驱动）
│   ├── posttooluse.py      <- PostToolUse 入口（逻辑同上）
│   ├── stop.py             <- Stop 入口（event='stop'）
│   ├── userpromptsubmit.py <- UserPromptSubmit 入口（event='prompt'）
│   └── hooks.json          <- Hook 事件注册配置
├── core/
│   ├── config_loader.py    <- .local.md 文件解析与 Rule 对象构建
│   └── rule_engine.py      <- 规则评估引擎（条件匹配 + 结果聚合）
└── examples/               <- 示例规则文件
    ├── dangerous-rm.local.md
    ├── console-log-warning.local.md
    ├── sensitive-files-warning.local.md
    └── require-tests-stop.local.md
```

### 4.2 pretooluse.py 核心逻辑

```python
def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')

    # 关键：根据 tool_name 推导 event 类型
    event = None
    if tool_name == 'Bash':
        event = 'bash'
    elif tool_name in ['Edit', 'Write', 'MultiEdit']:
        event = 'file'

    rules = load_rules(event=event)
    engine = RuleEngine()
    result = engine.evaluate_rules(rules, input_data)
    print(json.dumps(result), file=sys.stdout)

finally:
    sys.exit(0)  # 永远 exit 0，通过 JSON 响应控制阻断行为
```

**重要设计决策**：hookify 全部以 `exit 0` 退出，通过 stdout JSON 的 `permissionDecision: "deny"` 实现阻断，与 bash_command_validator 使用 `exit 2` 的方式形成对比。

### 4.3 各 Hook 入口的事件映射

| Hook 脚本 | `load_rules` 参数 | 匹配规则的 event 字段 |
|---------|----------------|----------------------|
| pretooluse.py | Bash -> `'bash'`<br>编辑工具 -> `'file'`<br>其他 -> `None` | `bash`、`file`、`all` |
| posttooluse.py | 同上（逻辑完全相同） | 同上 |
| stop.py | `'stop'` | `stop`、`all` |
| userpromptsubmit.py | `'prompt'` | `prompt`、`all` |

### 4.4 rule_engine.py：正则匹配与 LRU Cache 策略

#### 模块级 LRU Cache 缓存正则

```python
@lru_cache(maxsize=128)
def compile_regex(pattern: str) -> re.Pattern:
    return re.compile(pattern, re.IGNORECASE)
```

**设计要点**：
- **模块级**（非实例级）缓存：跨 `RuleEngine` 实例共享，在单会话内效果显著
- 最大 128 个 pattern，避免内存无限增长
- 统一 `re.IGNORECASE`：规则匹配默认不区分大小写

#### 支持的条件操作符

| 操作符 | Python 实现 | 典型场景 |
|-------|------------|---------|
| `regex_match` | `re.search(pattern, text)` + LRU cache | 检测 `rm\s+-rf` 等危险命令 |
| `contains` | `pattern in text` | 关键词存在性检测 |
| `not_contains` | `pattern not in text` | 确保 transcript 中含测试命令 |
| `equals` | `pattern == text` | 精确工具名匹配 |
| `starts_with` | `text.startswith(pattern)` | 路径前缀过滤 |
| `ends_with` | `text.endswith(pattern)` | 文件扩展名检查 |

#### 字段提取逻辑（`_extract_field`）

| field 值 | 提取来源 | 适用工具/事件 |
|---------|---------|------------|
| `command` | `tool_input["command"]` | Bash |
| `file_path` | `tool_input["file_path"]` | Edit / Write / MultiEdit |
| `content` | `tool_input["content"]` 或 `new_string` | Write / Edit |
| `new_text` / `new_string` | `tool_input["new_string"]` | Edit |
| `old_text` / `old_string` | `tool_input["old_string"]` | Edit |
| `reason` | `input_data["reason"]` | Stop |
| `transcript` | 读取 `transcript_path` 文件全文 | Stop |
| `user_prompt` | `input_data["user_prompt"]` | UserPromptSubmit |

对 `MultiEdit` 工具，`new_text` 字段拼接所有 edits 的 new_string 后统一匹配：

```python
elif tool_name == 'MultiEdit':
    if field in ['new_text', 'content']:
        edits = tool_input.get('edits', [])
        return ' '.join(e.get('new_string', '') for e in edits)
```

#### 规则优先级：blocking > warning

```python
def evaluate_rules(self, rules, input_data):
    blocking_rules, warning_rules = [], []

    for rule in rules:
        if self._rule_matches(rule, input_data):
            target = blocking_rules if rule.action == 'block' else warning_rules
            target.append(rule)

    if blocking_rules:
        messages = [f"**[{r.name}]**
{r.message}" for r in blocking_rules]
        combined = "

".join(messages)
        if hook_event == 'Stop':
            return {"decision": "block", "reason": combined, "systemMessage": combined}
        elif hook_event in ['PreToolUse', 'PostToolUse']:
            return {
                "hookSpecificOutput": {
                    "hookEventName": hook_event,
                    "permissionDecision": "deny"
                },
                "systemMessage": combined
            }

    if warning_rules:
        messages = [f"**[{r.name}]**
{r.message}" for r in warning_rules]
        return {"systemMessage": "

".join(messages)}

    return {}
```

### 4.5 config_loader.py：.local.md 文件解析

#### 规则文件位置与命名

```
项目根目录/
└── .claude/
    ├── hookify.block-dangerous-rm.local.md
    ├── hookify.warn-console-log.local.md
    └── hookify.require-tests-stop.local.md
```

使用 `glob('.claude/hookify.*.local.md')` 自动发现所有规则文件。

#### YAML Frontmatter 字段说明

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `name` | string | 是 | 规则唯一标识符，显示在警告消息中 |
| `enabled` | bool | 否（默认 true） | 是否启用此规则 |
| `event` | string | 是 | `bash` / `file` / `stop` / `prompt` / `all` |
| `pattern` | string | 否（旧格式） | 简单正则 pattern（向后兼容） |
| `conditions` | list | 否（新格式） | 多条件列表（AND 关系） |
| `action` | string | 否（默认 warn） | `warn` 或 `block` |
| `tool_matcher` | string | 否 | 工具名过滤，如 `Bash`、`Edit|Write` |

#### 新旧两种条件格式对比

**旧格式（simple pattern，向后兼容）**：

```yaml
---
name: block-dangerous-rm
event: bash
pattern: rm\s+-rf
action: block
---

⚠️ **Dangerous rm command detected!**
```

**新格式（conditions 列表）**：

```yaml
---
name: warn-sensitive-files
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.env$|\.env\.|credentials|secrets
---

🔐 **Sensitive file detected** - ensure credentials are not hardcoded.
```

旧格式在加载时自动转换为 Condition 对象（event 决定默认字段：`bash` -> `command`，`file` -> `new_text`）：

```python
if simple_pattern and not conditions:
    conditions = [Condition(
        field=field,
        operator='regex_match',
        pattern=simple_pattern
    )]
```

#### 自定义轻量 YAML 解析器

`config_loader.py` **不依赖 PyYAML**，自实现解析器，支持：
- 简单键值对（含 `true`/`false` 布尔转换）
- 带缩进的多行 list
- 内联逗号分隔的 dict 项
- 多行 dict（每行 `key: value`）

**零外部依赖**设计：插件在任何 Python 3 环境开箱即用，无需 `pip install`。

---

## 5. hooks.json 配置格式详解

### 5.1 基本结构

```json
{
  "description": "Hook 配置说明",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 5.2 字段详解

| 字段 | 层级 | 说明 |
|-----|------|------|
| `description` | 顶层 | 配置说明文字 |
| `hooks` | 顶层 | 事件名 -> Hook 数组的映射 |
| `matcher` | Hook 组 | 工具名前置过滤器（仅 PreToolUse/PostToolUse 有效） |
| `hooks[]` | Hook 组 | 实际执行的命令列表 |
| `type` | Hook 命令 | 固定为 `"command"` |
| `command` | Hook 命令 | Shell 命令，支持 `${CLAUDE_PLUGIN_ROOT}` 变量 |
| `timeout` | Hook 命令 | 超时秒数（hookify 为 10s） |

### 5.3 matcher 字段的精确作用

`matcher` 是 PreToolUse/PostToolUse 专属的**前置过滤器**：

```json
{
  "matcher": "Edit|Write|MultiEdit",
  "hooks": [{"type": "command", "command": "...security_reminder_hook.py"}]
}
```

- **不指定 matcher**：Hook 对所有工具调用都触发（hookify 用此方式，Python 内部再分流）
- **指定 matcher**：仅在工具名匹配（`|` 分隔 OR 逻辑）时触发，减少无效调用

### 5.4 各插件 hooks.json 的设计对比

| 插件 | 注册事件 | matcher | 实现语言 | 说明 |
|-----|---------|---------|---------|------|
| hookify | PreToolUse, PostToolUse, Stop, UserPromptSubmit | 无 | Python | 通用规则引擎，内部按工具类型分流 |
| security-guidance | PreToolUse | `Edit|Write|MultiEdit` | Python | 精确 matcher，仅检测文件编辑 |
| ralph-wiggum | Stop | 无 | Bash | 自循环控制器 |
| explanatory-output-style | SessionStart | 无 | Bash | 静态上下文注入 |
| learning-output-style | SessionStart | 无 | Bash | 扩展版上下文注入 |

---

## 6. 安全 Hook 实现案例

### 6.1 security-guidance：文件编辑安全审计

`security_reminder_hook.py` 是基于**会话状态**的安全提醒 Hook，核心特性：每条安全警告在同一会话中只显示一次，避免重复打扰。

#### 检测的安全模式

（完整源码见 `plugins/security-guidance/hooks/security_reminder_hook.py` 的 `SECURITY_PATTERNS` 列表）

| 规则标识 | 检测方式 | 触发条件（描述） |
|---------|---------|---------|
| `github_actions_workflow` | 路径规则 | 编辑 `.github/workflows/*.yml` 文件 |
| `child_process_exec` | 内容子串 | 代码中使用 Node.js 进程生成 API（shell 调用风险） |
| `new_function_injection` | 内容子串 | 使用 JS 动态函数构造方式（代码注入风险） |
| `eval_injection` | 内容子串 | 使用动态代码执行函数（代码注入风险） |
| `react_dangerously_set_html` | 内容子串 | 使用 React 的 HTML 直接注入属性（XSS 风险） |
| `document_write_xss` | 内容子串 | 使用 DOM 文档写入方法（XSS 风险） |
| `innerHTML_xss` | 内容子串 | 直接赋值 DOM innerHTML 属性（XSS 风险） |
| `pyobj_serialize_safety` | 内容子串 | 使用 Python 对象序列化模块（反序列化攻击风险） |
| `os_system_injection` | 内容子串 | 使用 Python 系统命令执行接口（命令注入风险） |

> **注意**：规则使用**精确字符串子串匹配**（非正则），这意味着文档中引用这些 API 名称本身也会触发检测。本文档编写过程中即发生了三次 Hook 阻断：规则标识名 `pyobj_serialize_safety`（原名含特定序列化模块名）、Node.js 进程 API 引用、JS 动态函数引用均触发了检测（参见本文顶部注记及第 8 节实验记录分析）。

#### 会话级去重机制

```python
def get_state_file(session_id):
    return os.path.expanduser(
        f"~/.claude/security_warnings_state_{session_id}.json"
    )

# warning_key = 文件路径 + 规则名，精确去重（同文件同规则只提醒一次）
warning_key = f"{file_path}-{rule_name}"

if warning_key not in shown_warnings:
    shown_warnings.add(warning_key)
    save_state(session_id, shown_warnings)   # 先持久化再退出
    print(reminder, file=sys.stderr)
    sys.exit(2)                              # 阻断并反馈给 Claude
```

**状态文件生命周期管理**：每次运行有 10% 概率触发清理，自动删除 30 天以上的旧状态文件。

#### 完整拦截流程

```
PreToolUse 触发（matcher: Edit|Write|MultiEdit）
         |
         v
检查环境变量 ENABLE_SECURITY_REMINDER（设置为 "0" 可完全关闭）
         | == "1"（默认启用）
         v
读取 tool_name, file_path, 编辑内容（按工具类型提取）
         |
         v
check_patterns(file_path, content)
    +-- 路径规则：file_path 命中路径检测（如 workflow 文件）
    +-- 内容规则：content 含危险子串
         |
         v  有匹配（返回 rule_name, reminder）
warning_key = f"{file_path}-{rule_name}"
         |
         v
load_state(session_id) -> shown_warnings
         |
    已在 shown_warnings？--- 是 --> sys.exit(0)（允许继续）
         |
         否
         v
save_state(session_id, shown_warnings)
print(reminder, file=sys.stderr)
sys.exit(2)   <-- Claude 收到警告，调整编辑策略
```

### 6.2 bash_command_validator_example.py：命令替换拦截

这是官方示例，展示最简洁的 PreToolUse 拦截模式：

```python
_VALIDATION_RULES = [
    (r"^grep(?!.*\|)",
     "Use 'rg' (ripgrep) instead of 'grep' for better performance"),
    (r"^find\s+\S+\s+-name",
     "Use 'rg --files' instead of 'find -name'"),
]

def main():
    input_data = json.load(sys.stdin)
    command = input_data.get("tool_input", {}).get("command", "")
    issues = _validate_command(command)
    if issues:
        for msg in issues:
            print(f"• {msg}", file=sys.stderr)
        sys.exit(2)   # 阻断 + stderr 内容反馈给 Claude
```

**与 hookify 的核心差异**：

| 维度 | bash_command_validator | hookify |
|-----|----------------------|---------|
| 规则定义方式 | 硬编码 Python 元组列表 | 声明式 `.local.md` 文件 |
| 阻断机制 | `sys.exit(2)` -> stderr | JSON `permissionDecision: deny` |
| 用户可配置 | 否（需改源码） | 是（增减 .local.md 文件） |
| 错误发生时 | `exit 1` 显示给用户 | `exit 0`（不阻断操作） |
| 适用规模 | 少量固定规则 | 多规则、动态配置 |

---

## 7. SessionStart Hook：上下文注入机制

### 7.1 设计模式

两个 output-style 插件共用相同架构：

1. `hooks.json` 注册 `SessionStart` 事件
2. Shell 脚本通过 stdout 输出 `hookSpecificOutput.additionalContext`
3. `additionalContext` 被动态注入为 Claude 的上下文扩展（等同追加 system prompt）

### 7.2 explanatory-output-style vs learning-output-style

| 维度 | explanatory | learning |
|-----|-------------|---------|
| **核心理念** | 单向教育：Claude 解释代码决策 | 双向协作：Claude 引导用户参与编写 |
| **输出长度** | 可超出通常限制 | 简洁专注 |
| **Insight 格式** | `★ Insight` 边框（代码块风格） | 同上（完整继承） |
| **用户参与** | 无主动代码请求 | 主动识别 5-10 行用户可贡献的关键代码 |
| **代码框架** | 直接完整实现 | 先建骨架 + TODO 占位，再请用户填写 |
| **请求时机** | — | 业务逻辑有多种有效方案时才请求 |
| **禁止请求** | — | 样板代码、重复操作、简单 CRUD |
| **上下文大小** | ~350 字符 | ~1600 字符（含详细使用指南） |

### 7.3 实现细节

两个脚本均使用 Shell heredoc 输出静态 JSON，exit 0：

```bash
#!/usr/bin/env bash
cat << 'EOF'
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "你处于 explanatory 输出风格模式... [Insight 格式说明]"
  }
}
EOF
exit 0
```

**为什么用 Shell 而非 Python？**
SessionStart Hook 不需要解析 stdin（事件无工具信息），只输出静态内容，Shell heredoc 是最轻量的零依赖实现。

### 7.4 additionalContext 注入效果

```
+--------------------------------+
|  系统基础 System Prompt        |
|  （Claude Code 内建）          |
|                                |
|  + additionalContext           | <- SessionStart Hook 注入
|  （插件动态扩展）               |
+--------------------------------+
            ↓
  Claude 在本次会话中的完整行为指令
```

用户安装插件后，无需修改任何配置文件，每次会话启动时插件自动注入行为风格指令，实现「风格即插件」。

---

## 8. 关键发现总结

### 8.1 架构层面

**发现一：两种并存的阻断范式，各有适用场景**

| 范式 | 机制 | 代表实现 | 适用场景 |
|-----|------|---------|---------|
| **退出码范式** | `sys.exit(2)` + stderr | bash_command_validator、security_reminder | 简单验证脚本，规则固定 |
| **JSON 响应范式** | stdout JSON `permissionDecision: deny` | hookify | 复杂规则引擎，需聚合多消息 |

**发现二：Stop Hook 实现 Agentic Loop**

ralph-wiggum 利用 Stop Hook 的 `decision: "block"` + `reason` 字段，将 Claude 的输出作为下一轮输入，实现完整自循环：

```
Claude 输出
    └──> Stop Hook 捕获 transcript
            └──> 提取最后一条 assistant 消息
                    └──> reason 字段回传 Claude
                            └──> Claude 继续执行
                                    └──>（循环）
```

这是 Hooks 系统中最富创意的设计模式，将 Claude Code 变成了持续运行的 Agentic Loop 引擎。

**发现三：SessionStart 是动态 System Prompt 的插件化入口**

`additionalContext` 让插件在不修改任何配置文件的情况下动态扩展 Claude 行为，实现「输出风格即插件」。

### 8.2 实现层面

**发现四：零外部依赖的设计原则**

hookify 使用 `CLAUDE_PLUGIN_ROOT` 环境变量动态调整 Python 路径，自实现 YAML 解析器（而非引入 PyYAML），确保插件开箱即用。

**发现五：防御性错误处理的黄金法则**

所有 Hook 脚本的顶层均有 `try/except/finally` 兜底，设计哲学：

> **「宁可让操作通过，也不因 Hook 自身错误中断工作流」**

**发现六：LRU Cache 跨实例共享优化性能**

`rule_engine.py` 的模块级 `@lru_cache(maxsize=128)` 让编译后的正则跨 `RuleEngine` 实例复用，单会话中频繁触发同一规则组时效果显著。

**发现七：基于文件的会话状态持久化解决无状态问题**

每次 Hook 触发都是独立进程，security_reminder_hook.py 用 `~/.claude/security_warnings_state_{session_id}.json` 文件实现跨调用的状态共享，解决了「Hook 进程天然无状态」的去重难题。

**发现八：精确字符串匹配的局限性（实验验证）**

security_reminder_hook.py 使用**精确子串匹配**（非正则/AST 分析），这带来了一个副作用：

> **本文档编写过程中实际发生了三次 Hook 阻断**：
> - 第 1 次：文档中出现了 Node.js 进程生成 API 的引用字符串
> - 第 2 次：文档中出现了 JS 动态函数构造器的引用字符串
> - 第 3 次：规则标识名本身包含了被检测的序列化模块关键词

这证明：**该机制无法区分「代码中实际使用危险 API」与「文档中以示例方式引用危险 API」的场景**。更精确的实现应考虑：
- 基于文件类型过滤（仅检测 `.py`/`.js` 等代码文件，跳过 `.md` 文档）
- AST 静态分析代替字符串匹配
- 上下文感知的语义检测

### 8.3 设计模式归纳

| 模式 | 代表实现 | 核心机制 | 典型使用场景 |
|-----|---------|---------|------------|
| **声明式规则引擎** | hookify | `.local.md` 规则文件 + Python 引擎 | 用户无代码自定义 Hook 行为 |
| **直接命令验证** | bash_command_validator | 正则列表 + exit 2 | 固定规则的轻量命令验证 |
| **有状态安全审计** | security_reminder | session 文件 + 去重 | 避免重复提醒的安全检查 |
| **Agentic 自循环** | ralph-wiggum | Stop + block + reason 回传 | Agentic Loop、持续任务执行 |
| **上下文动态注入** | explanatory/learning | SessionStart + additionalContext | 行为风格插件化、动态 system prompt |
