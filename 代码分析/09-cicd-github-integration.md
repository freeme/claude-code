# Claude Code · CI/CD 与 GitHub 集成深度分析

> 分析对象：`/Volumes/Data/agent-frameworks/claude-code`
> 覆盖范围：`.github/workflows/`（12 个 YAML）、`scripts/`（8 个脚本）、`.github/ISSUE_TEMPLATE/`（5 个模板）

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [claude.yml —— Claude 介入 GitHub Actions](#2-claudeyml--claude-介入-github-actions)
3. [Issue 生命周期自动化](#3-issue-生命周期自动化)
4. [TypeScript 脚本深度分析](#4-typescript-脚本深度分析)
5. [Shell 脚本工具层](#5-shell-脚本工具层)
6. [Issue 模板体系](#6-issue-模板体系)
7. [各 Workflow 触发条件对比](#7-各-workflow-触发条件对比)
8. [Issue 完整生命周期流程图](#8-issue-完整生命周期流程图)
9. [AI 参与程度边界分析](#9-ai-参与程度边界分析)
10. [Label 系统设计](#10-label-系统设计)
11. [关键发现总结](#11-关键发现总结)

---

## 1. 整体架构概览

Claude Code 仓库构建了一套**高度自动化的 Issue 治理体系**，将 AI 判断与规则引擎深度融合。整体分为三层：

```
┌──────────────────────────────────────────────────────────────────────┐
│                        GitHub Events Layer                           │
│   issues.*  │  issue_comment.*  │  pull_request.*  │  schedule       │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ 事件触发
┌──────────────────────────▼───────────────────────────────────────────┐
│                      Workflow Orchestration Layer                     │
│                                                                      │
│  ┌────────────────┐  ┌──────────────────┐  ┌───────────────────┐    │
│  │  AI-Powered    │  │  Rule-Based      │  │  Scheduled Batch  │    │
│  │  Workflows     │  │  Workflows       │  │  Workflows        │    │
│  │                │  │                  │  │                   │    │
│  │ claude.yml     │  │ issue-lifecycle- │  │ sweep.yml         │    │
│  │ claude-issue-  │  │ comment.yml      │  │ auto-close-       │    │
│  │ triage.yml     │  │ remove-autoclose │  │ duplicates.yml    │    │
│  │ claude-dedupe- │  │ lock-closed-     │  │ lock-closed-      │    │
│  │ issues.yml     │  │ issues.yml       │  │ issues.yml        │    │
│  └────────────────┘  └──────────────────┘  └───────────────────┘    │
│                                                                      │
│  ┌────────────────┐  ┌──────────────────┐                           │
│  │  Dispatch/     │  │  Observability   │                           │
│  │  Routing       │  │  Workflows       │                           │
│  │                │  │                  │                           │
│  │ issue-opened-  │  │ log-issue-       │                           │
│  │ dispatch.yml   │  │ events.yml       │                           │
│  │ backfill-      │  │                  │                           │
│  │ duplicate-     │  │                  │                           │
│  │ comments.yml   │  │                  │                           │
│  └────────────────┘  └──────────────────┘                           │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ 调用
┌──────────────────────────▼───────────────────────────────────────────┐
│                       Script Execution Layer                          │
│                                                                      │
│  TypeScript (Bun)               Shell                                │
│  ├── auto-close-duplicates.ts   ├── gh.sh (安全封装 gh CLI)           │
│  ├── lifecycle-comment.ts       ├── edit-issue-labels.sh             │
│  ├── sweep.ts                   └── comment-on-duplicates.sh         │
│  ├── backfill-duplicate-                                             │
│  │   comments.ts                                                     │
│  └── issue-lifecycle.ts (配置)                                       │
└──────────────────────────────────────────────────────────────────────┘
```

**关键依赖：**
- AI 核心：`anthropics/claude-code-action@v1`（封装了 Claude API 调用）
- 脚本运行时：Bun（TypeScript 原生执行，无需编译）
- 可观测性：Statsig（事件埋点，用于分析 Issue 处理效率）

---

## 2. claude.yml —— Claude 介入 GitHub Actions

### 2.1 触发机制

```yaml
# claude.yml
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]
```

**触发过滤条件**（`if` 表达式）：所有触发事件均需要内容中包含 `@claude` 关键字才会实际执行 Job：

```yaml
if: |
  (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
  (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
```

**设计要点：**
- `@claude` 是唯一触发词，避免 Bot 误触发
- 支持 Issue 正文、Issue 标题、评论、PR Review 评论、PR Review 主体五个入口
- `issues: [opened, assigned]` 表示 Issue 被 assign 给某人时也可通过 `@claude` 触发

### 2.2 权限设计（最小权限原则）

```yaml
permissions:
  contents: read          # 读取代码
  pull-requests: read     # 读取 PR 信息
  issues: read            # 读取 Issue 信息
  id-token: write         # OIDC 身份验证
```

注意：此 workflow **没有 `issues: write` 权限**，Claude 在此模式下只能以只读方式响应，不能直接操作 Issue（写操作由其他 workflow 完成）。

### 2.3 Claude Action 配置

```yaml
- name: Run Claude Code
  id: claude
  uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    claude_args: "--model claude-sonnet-4-5-20250929"
```

- 使用 `claude-sonnet-4-5-20250929` 模型（高性价比的 Sonnet）
- 不设置 `prompt`，意味着 Claude 会直接读取 GitHub 上下文（Issue/PR/Comment 内容）自动生成响应

### 2.4 与其他 AI Workflow 的对比

| Workflow | 模型 | 触发方式 | prompt 方式 | 权限 |
|---------|------|---------|------------|------|
| `claude.yml` | claude-sonnet-4-5-20250929 | `@claude` 提及 | 自动读取上下文 | 只读 |
| `claude-issue-triage.yml` | claude-opus-4-6 | Issue 开启/评论 | 固定 `/triage-issue` slash command | 写（issues） |
| `claude-dedupe-issues.yml` | claude-sonnet-4-5-20250929 | Issue 开启 | 固定 `/dedupe` slash command | 写（issues） |

---

## 3. Issue 生命周期自动化

### 3.1 claude-issue-triage.yml —— AI 自动分类

**触发条件：**
```yaml
on:
  issues:
    types: [opened]       # Issue 新建时立即触发
  issue_comment:
    types: [created]      # 新评论时重新分类
```

**并发控制（关键）：**
```yaml
concurrency:
  group: issue-triage-${{ github.event.issue.number }}
  cancel-in-progress: true
```

每个 Issue 同一时间只跑一个分类任务，新触发会取消旧任务，防止并发冲突。

**AI 分类实现：**
```yaml
env:
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  GH_REPO: ${{ github.repository }}
  CLAUDE_CODE_SCRIPT_CAPS: '{"edit-issue-labels.sh":2}'   # 限制脚本调用次数

with:
  allowed_non_write_users: "*"    # 允许所有用户触发（包括非 write 权限）
  prompt: "/triage-issue REPO: ${{ github.repository }} ISSUE_NUMBER: ${{ github.event.issue.number }} EVENT: ${{ github.event_name }}"
  claude_args: "--model claude-opus-4-6"   # 使用最强模型分类
```

**关键设计：**
- 使用 `claude-opus-4-6`（最强模型）确保分类准确
- `CLAUDE_CODE_SCRIPT_CAPS: '{"edit-issue-labels.sh":2}'` 限制标签编辑脚本最多调用 2 次，防止 AI 无限循环修改标签
- `/triage-issue` 是 slash command，Claude 会按预定义的分类规则执行
- `allowed_non_write_users: "*"` 允许任何人提交的 Issue 都能触发分类

**安全审计：** `non-write-users-check.yml` 会在 PR 修改 `.github/**` 文件时，检查是否有新增 `allowed_non_write_users`，并自动发出安全警告注释。

### 3.2 claude-dedupe-issues.yml —— AI 重复检测

**触发：**
```yaml
on:
  issues:
    types: [opened]      # 每个新 Issue 都自动检查重复
  workflow_dispatch:
    inputs:
      issue_number:      # 手动指定 Issue 进行回填
```

**AI 去重指令：**
```yaml
prompt: "/dedupe ${{ github.repository }}/issues/${{ github.event.issue.number || inputs.issue_number }}"
```

**执行能力限制：**
```yaml
env:
  CLAUDE_CODE_SCRIPT_CAPS: '{"comment-on-duplicates.sh":1}'  # 只能发一次重复评论
```

**Statsig 埋点：** 去重完成后自动上报 `github_duplicate_comment_added` 事件：

```yaml
- name: Log duplicate comment event to Statsig
  if: always()
  run: |
    EVENT_PAYLOAD=$(jq -n \
      --arg issue_number "$ISSUE_NUMBER" \
      --arg repo "$REPO" \
      '{
        events: [{
          eventName: "github_duplicate_comment_added",
          value: 1,
          metadata: { repository: $repo, issue_number: ... }
        }]
      }')
    curl -X POST https://events.statsigapi.net/v1/log_event ...
```

### 3.3 auto-close-duplicates.yml —— 定时批量关闭

**触发：** 每天 UTC 09:00 定时运行（北京时间 17:00）

```yaml
on:
  schedule:
    - cron: "0 9 * * *"
  workflow_dispatch:
```

**执行逻辑：** 调用 `bun run scripts/auto-close-duplicates.ts`，详见第 4.1 节。

### 3.4 issue-lifecycle-comment.yml —— 标签触发通知

```yaml
on:
  issues:
    types: [labeled]    # 任意 Label 添加时触发
```

当 Issue 被添加生命周期 Label（`invalid`/`needs-repro`/`needs-info`/`stale`/`autoclose`）时，自动发送包含"倒计时"的通知评论，让作者知晓剩余时间。

### 3.5 sweep.yml —— 定时生命周期检查

**触发：** 每天 UTC 10:00 和 22:00 各运行一次（一天两次）

```yaml
on:
  schedule:
    - cron: "0 10,22 * * *"
```

**并发保护：**
```yaml
concurrency:
  group: daily-issue-sweep   # 防止两个定时任务重叠
```

### 3.6 remove-autoclose-label.yml —— 用户活动撤销自动关闭

```yaml
on:
  issue_comment:
    types: [created]

jobs:
  remove-autoclose:
    if: |
      github.event.issue.state == 'open' &&
      contains(github.event.issue.labels.*.name, 'autoclose') &&
      github.event.comment.user.login != 'github-actions[bot]'
```

当带有 `autoclose` 标签的 Issue 收到**非 Bot 评论**时，自动移除 `autoclose` 标签，重置生命周期计时。

### 3.7 lock-closed-issues.yml —— 关闭 7 天后锁定

**触发：** 每天 UTC 14:00（太平洋时间 8 AM）

核心逻辑（内嵌 JavaScript）：
```javascript
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

// 按 updated_at 升序排序，找到超过 7 天未更新的已关闭 Issue
const { data: issues } = await github.rest.issues.listForRepo({
  state: 'closed',
  sort: 'updated',
  direction: 'asc',   // 最旧的先处理
  per_page: 100,
  page: page
});

// 当遇到 updated_at > 7 天前的 Issue 时停止（因为是升序，后面的只会更新）
if (updatedAt > sevenDaysAgo) {
  hasMore = false;
  break;
}
```

### 3.8 issue-opened-dispatch.yml —— 外部系统通知

```yaml
on:
  issues:
    types: [opened]

steps:
  - name: Process new issue
    run: |
      gh api repos/${TARGET_REPO}/dispatches \
        -f event_type=issue_opened \
        -f client_payload[issue_url]="${ISSUE_URL}" || {
          exit 0   # 失败时静默退出，不影响主流程
        }
```

将新 Issue 事件 dispatch 到外部私有仓库（通过 `ISSUE_OPENED_DISPATCH_TARGET_REPO` Secret），用于内部监控或通知系统。

### 3.9 log-issue-events.yml —— Statsig 数据采集

```yaml
on:
  issues:
    types: [opened, closed]
```

记录 Issue 开启和关闭事件到 Statsig，为产品分析提供数据基础。注意代码有 **注入防护设计**：

```bash
# 所有值通过环境变量传递，防止 Shell 注入
ISSUE_TITLE: ${{ github.event.issue.title }}
# 不直接在 JSON 字符串中嵌入 ${{ }} 模板
```

### 3.10 backfill-duplicate-comments.yml —— 历史数据回填

**触发：** 仅手动触发（`workflow_dispatch`）

```yaml
inputs:
  days_back:
    description: 'How many days back to look for old issues'
    default: '90'
  dry_run:
    description: 'Dry run mode'
    default: 'true'   # 默认安全模式
```

用于给存量 Issue 补充重复检测，通过 `claude-dedupe-issues.yml` 的 `workflow_dispatch` API 批量触发。

---

## 4. TypeScript 脚本深度分析

### 4.1 auto-close-duplicates.ts —— 重复 Issue 自动关闭

**核心算法（非 AI 判断，纯规则）：**

```
判断一个 Issue 是否应该被自动关闭为重复，需同时满足：
1. Issue 已存在 ≥3 天
2. 存在 Bot 发的包含 "Found * possible duplicate" 的评论
3. 该重复检测评论距今已 ≥3 天
4. 重复检测评论之后无任何新评论（即无后续活动）
5. Issue 作者没有对重复检测评论点 👎（即未反对）
6. 能从评论中提取出具体的重复 Issue 编号
```

**重复 Issue 号提取（双格式支持）：**

```typescript
function extractDuplicateIssueNumber(commentBody: string): number | null {
  // 格式1：#123
  let match = commentBody.match(/#(\d+)/);
  if (match) return parseInt(match[1], 10);
  
  // 格式2：https://github.com/owner/repo/issues/123
  match = commentBody.match(/github\.com\/[^\/]+\/[^\/]+\/issues\/(\d+)/);
  if (match) return parseInt(match[1], 10);
  
  return null;
}
```

**关闭动作：**

```typescript
async function closeIssueAsDuplicate(owner, repo, issueNumber, duplicateOfNumber, token) {
  // 1. 更新 Issue 状态
  await githubRequest(`/repos/${owner}/${repo}/issues/${issueNumber}`, token, 'PATCH', {
    state: 'closed',
    state_reason: 'duplicate',   // GitHub 原生重复关闭原因
    labels: ['duplicate']
  });

  // 2. 发送关闭说明评论
  await githubRequest(`/repos/${owner}/${repo}/issues/${issueNumber}/comments`, token, 'POST', {
    body: `This issue has been automatically closed as a duplicate of #${duplicateOfNumber}.\n\n...🤖 Generated with [Claude Code](https://claude.ai/code)`
  });
}
```

**分页处理（防止大仓库超时）：**
```typescript
// 最多处理 20 页，每页 100 条，最多 2000 个 Issue
if (page > 20) break;
```

**用户反对机制（👎 保护）：**
```typescript
const authorThumbsDown = reactions.some(
  (reaction) =>
    reaction.user.id === issue.user.id &&   // 必须是 Issue 作者
    reaction.content === "-1"                // 点的是 👎
);
if (authorThumbsDown) continue;  // 跳过，不关闭
```

### 4.2 lifecycle-comment.ts —— 生命周期通知

**最简的事件驱动设计：**

```typescript
// 从 issue-lifecycle.ts 读取配置（单一真相来源）
const entry = lifecycle.find((l) => l.label === label);
if (!entry) process.exit(0);  // 非生命周期 Label，静默退出

const body = `${entry.nudge} This issue will be closed automatically if there's no activity within ${entry.days} days.`;
```

评论内容完全由 `issue-lifecycle.ts` 配置驱动，逻辑与配置完全解耦。

### 4.3 sweep.ts —— 批量生命周期执行

**两阶段执行：**

**阶段一：markStale() —— 标记过期 Issue**
```typescript
async function markStale(owner: string, repo: string) {
  const staleDays = lifecycle.find((l) => l.label === "stale")!.days;  // 14 天
  
  for (const issue of issues) {
    if (issue.pull_request) continue;     // 跳过 PR
    if (issue.locked) continue;           // 跳过已锁定
    if (issue.assignees?.length > 0) continue;  // 跳过已分配的（有人跟进）

    // 高 upvote 保护：>=10 个 👍 不标记 stale
    const thumbsUp = issue.reactions?.["+1"] ?? 0;
    if (thumbsUp >= STALE_UPVOTE_THRESHOLD) continue;
    
    await githubRequest(`${base}/labels`, "POST", { labels: ["stale"] });
  }
}
```

**阶段二：closeExpired() —— 关闭超时 Issue**
```typescript
async function closeExpired(owner: string, repo: string) {
  for (const { label, days, reason } of lifecycle) {
    // 查询带特定 Label 的 Issue
    const issues = await githubRequest(`/issues?labels=${label}&...`);
    
    for (const issue of issues) {
      // 安全检查：Label 打上后是否有人工评论？
      const hasHumanComment = comments.some(
        (c) => c.user && c.user.type !== "Bot"
      );
      if (hasHumanComment) continue;  // 有人跟进则跳过
      
      // 发关闭评论 + 关闭 Issue
      await githubRequest(`${base}/comments`, "POST", { body: CLOSE_MESSAGE(reason) });
      await githubRequest(base, "PATCH", { state: "closed", state_reason: "not_planned" });
    }
  }
}
```

**关闭消息格式：**
```typescript
const CLOSE_MESSAGE = (reason: string) =>
  `Closing for now — ${reason}. Please [open a new issue](${NEW_ISSUE}) if this is still relevant.`;
```

### 4.4 issue-lifecycle.ts —— 生命周期配置（单一真相来源）

```typescript
export const lifecycle = [
  {
    label: "invalid",
    days: 3,
    reason: "this doesn't appear to be about Claude Code",
    nudge: "This doesn't appear to be about [Claude Code](...). For general Anthropic support, visit [support.anthropic.com](...)",
  },
  {
    label: "needs-repro",
    days: 7,
    reason: "we still need reproduction steps to investigate",
    nudge: "We weren't able to reproduce this...",
  },
  {
    label: "needs-info",
    days: 7,
    reason: "we still need a bit more information to move forward",
    nudge: "We need more information to continue investigating...",
  },
  {
    label: "stale",
    days: 14,
    reason: "inactive for too long",
    nudge: "This issue has been automatically marked as stale due to inactivity.",
  },
  {
    label: "autoclose",
    days: 14,
    reason: "inactive for too long",
    nudge: "This issue has been marked for automatic closure.",
  },
] as const;

export const STALE_UPVOTE_THRESHOLD = 10;  // 10 个 👍 保护 Issue 不被 stale
```

**设计亮点：**
- `as const` 确保类型安全，TypeScript 可以推导出具体的 label 字符串联合类型
- 所有超时时间、关闭原因、nudge 消息集中管理
- `lifecycle-comment.ts` 和 `sweep.ts` 都导入此配置，修改一处全局生效

### 4.5 backfill-duplicate-comments.ts —— 历史回填

核心逻辑：对范围内没有重复检测评论的 Issue，通过 GitHub Actions API 触发 `claude-dedupe-issues.yml`：

```typescript
async function triggerDedupeWorkflow(owner, repo, issueNumber, token, dryRun) {
  await githubRequest(
    `/repos/${owner}/${repo}/actions/workflows/claude-dedupe-issues.yml/dispatches`,
    token, 'POST',
    {
      ref: 'main',
      inputs: { issue_number: issueNumber.toString() }
    }
  );
}

// 触发间隔 1 秒，避免 API Rate Limit
await new Promise(resolve => setTimeout(resolve, 1000));
```

---

## 5. Shell 脚本工具层

### 5.1 gh.sh —— 安全封装的 gh CLI

**安全设计（白名单模式）：**

```bash
# 仅允许 4 类命令
case "$CMD" in
  "issue view"|"issue list"|"search issues"|"label list")
    ;;
  *)
    echo "Error: only 'issue view', 'issue list', 'search issues', 'label list' are allowed" >&2
    exit 1
    ;;
esac

# 仅允许 4 类 flag
ALLOWED_FLAGS=(--comments --state --limit --label)

# 搜索查询防注入：禁止 repo:/org:/user: 限定符
if [[ "$QUERY_LOWER" == *"repo:"* || "$QUERY_LOWER" == *"org:"* || "$QUERY_LOWER" == *"user:"* ]]; then
  echo "Error: search query must not contain repo:, org:, or user: qualifiers" >&2
  exit 1
fi
```

**用途：** 供 Claude AI 在 triage 和 dedupe workflow 中调用，通过严格的白名单防止 AI 执行危险的 gh 命令。

### 5.2 edit-issue-labels.sh —— 标签操作

```bash
# Issue 编号从 GITHUB_EVENT_PATH 读取，与触发事件绑定
ISSUE=$(jq -r '.issue.number // .inputs.issue_number // empty' "${GITHUB_EVENT_PATH:?}")

# 操作前验证标签是否存在于仓库
VALID_LABELS=$(gh label list --limit 500 --json name --jq '.[].name')
for label in "${ADD_LABELS[@]}"; do
  if echo "$VALID_LABELS" | grep -qxF "$label"; then
    FILTERED_ADD+=("$label")
  fi
done
```

**安全设计：** Issue 编号从事件 payload 读取而非参数，防止 AI 操作错误的 Issue。

### 5.3 comment-on-duplicates.sh —— 发重复评论

```bash
# 最多 3 个候选重复 Issue
if [[ ${#DUPLICATES[@]} -gt 3 ]]; then
  echo "Error: --potential-duplicates accepts at most 3 issues" >&2
  exit 1
fi

# 验证所有 Issue 都存在
for dup in "${DUPLICATES[@]}"; do
  if ! gh issue view "$dup" --repo "$REPO" &>/dev/null; then
    echo "Error: issue #$dup does not exist in $REPO" >&2
    exit 1
  fi
done
```

**生成的评论格式：**
```
Found N possible duplicate issues:

1. https://github.com/anthropics/claude-code/issues/456
2. https://github.com/anthropics/claude-code/issues/789

This issue will be automatically closed as a duplicate in 3 days.

- If your issue is a duplicate, please close it and 👍 the existing issue instead
- To prevent auto-closure, add a comment or 👎 this comment

🤖 Generated with Claude Code
```

---

## 6. Issue 模板体系

### 6.1 模板类型

| 模板 | 用途 | 预设 Label |
|-----|------|-----------|
| `bug_report.yml` | 功能缺陷报告 | `bug` |
| `feature_request.yml` | 新功能请求 | `enhancement` |
| `documentation.yml` | 文档问题 | `documentation` |
| `model_behavior.yml` | AI 行为异常 | `model` |
| `config.yml` | 关闭空白 Issue + 社区链接 | - |

### 6.2 bug_report.yml 设计要点

**Preflight Checklist（提交前强制确认）：**
```yaml
- type: checkboxes
  id: preflight
  options:
    - label: I have searched existing issues and this hasn't been reported yet
      required: true    # 强制确认，减少重复
    - label: This is a single bug report
      required: true
    - label: I am using the latest version of Claude Code
      required: true
```

**关键信息字段：**
- 版本号（`claude --version`）—— 辅助 triage
- 操作系统 + 终端类型 —— 平台问题定位
- 是否 Regression + 最后正常版本 —— 加速 bisect 排查
- API 平台（Anthropic/Bedrock/Vertex）—— 平台特有问题分类

### 6.3 model_behavior.yml 的特殊性

`model_behavior.yml` 专门处理 AI 行为异常，字段设计与 bug 报告不同：
- **行为类型**：修改了不该改的文件 / 忽略指令 / 无故恢复更改等
- **精确的步骤描述**：你让 Claude 做什么 / Claude 实际做了什么 / 期望行为
- **权限模式**：Accept Edits ON/OFF —— 影响行为的关键变量
- **影响的文件列表**：包括意外访问的文件（安全相关）

### 6.4 config.yml 的社区引导设计

```yaml
blank_issues_enabled: false    # 禁止空白 Issue，强制使用模板
contact_links:
  - name: 💬 Discord Community
    url: https://anthropic.com/discord
  - name: 📖 Documentation
    url: https://docs.claude.com/en/docs/claude-code
```

将非代码问题引导到 Discord 和文档，减少 Issue 噪音。

---

## 7. 各 Workflow 触发条件对比

| Workflow | 触发事件 | 触发条件 | 执行频率 | 超时 |
|---------|---------|---------|---------|------|
| `claude.yml` | issue_comment / pr_review_comment / issues / pr_review | 内容含 `@claude` | 按需 | 无 |
| `claude-issue-triage.yml` | issues[opened] / issue_comment[created] | Issue 非 PR + 评论非 Bot | 每个新 Issue + 每条人工评论 | 10 分钟 |
| `claude-dedupe-issues.yml` | issues[opened] / workflow_dispatch | 每个新 Issue | 每个新 Issue | 10 分钟 |
| `auto-close-duplicates.yml` | schedule / workflow_dispatch | UTC 09:00 每天 | 1 次/天 | 10 分钟 |
| `issue-lifecycle-comment.yml` | issues[labeled] | 任意 Label 添加 | 每次打标签 | 无 |
| `sweep.yml` | schedule / workflow_dispatch | UTC 10:00 + 22:00 | 2 次/天 | 无 |
| `lock-closed-issues.yml` | schedule / workflow_dispatch | UTC 14:00 每天 | 1 次/天 | 无 |
| `remove-autoclose-label.yml` | issue_comment[created] | Issue 开启 + 有 autoclose 标签 + 非 Bot 评论 | 按需 | 无 |
| `log-issue-events.yml` | issues[opened, closed] | 所有 Issue 开启/关闭 | 每个 Issue 事件 | 无 |
| `issue-opened-dispatch.yml` | issues[opened] | 所有新 Issue | 每个新 Issue | 1 分钟 |
| `backfill-duplicate-comments.yml` | workflow_dispatch | 手动触发 | 按需（一次性） | 30 分钟 |
| `non-write-users-check.yml` | pull_request[paths: .github/**] | PR 修改 .github 文件 | 每个相关 PR | 无 |

---

## 8. Issue 完整生命周期流程图

```
用户提交 Issue
      │
      ├──────────────────────────────────────────────────────────────┐
      │                                                              │
      ▼                                                              ▼
[issue-opened-dispatch.yml]                              [log-issue-events.yml]
 dispatch 到内部系统                                      记录 opened 事件到 Statsig
      │
      │ 并行触发（同一 issues[opened] 事件）
      ├─────────────────────────────────┐
      │                                 │
      ▼                                 ▼
[claude-issue-triage.yml]       [claude-dedupe-issues.yml]
 claude-opus-4-6 分类             claude-sonnet 检测重复
 调用 edit-issue-labels.sh        调用 comment-on-duplicates.sh
 打上分类 Label                   ├─ 无重复 → 跳过
      │                           └─ 有重复 → 发 "Found N possible duplicates" 评论
      │                                            │
      │                                            │ 3 天后（每天 09:00 运行）
      │                                            ▼
      │                                   [auto-close-duplicates.yml]
      │                                    auto-close-duplicates.ts
      │                                    检查：有无后续评论？有无 👎？
      │                                    ├─ 有 👎 或 有评论 → 跳过
      │                                    └─ 无 → 关闭 Issue (state_reason: duplicate)
      │
      ▼
[Issue 分类完成，带有 Label]
      │
      ├── Label = "invalid" → [issue-lifecycle-comment.yml]
      │                        发通知：3 天内无响应将关闭
      │
      ├── Label = "needs-repro" → [issue-lifecycle-comment.yml]
      │                            发通知：7 天内无响应将关闭
      │
      ├── Label = "needs-info" → [issue-lifecycle-comment.yml]
      │                           发通知：7 天内无响应将关闭
      │
      └── 无生命周期 Label → 正常开启状态
                │
                │ 14 天无更新（每天 10:00/22:00 运行）
                ▼
         [sweep.yml] → sweep.ts: markStale()
          检查 👍 < 10，无 assignee，未锁定
          → 打 "stale" Label
                │
                ▼
         [issue-lifecycle-comment.yml]
          发通知：14 天内无响应将关闭
                │
                │ 用户有新评论？
                ├──── 是 ──→ [remove-autoclose-label.yml]
                │              移除 autoclose/stale 标签
                │              → Issue 重置，继续活跃
                │
                │ 继续无活动 14 天（sweep.ts: closeExpired()）
                ▼
         检查 Label 打上后是否有人工评论
         ├─ 有 → 跳过（安全网）
         └─ 无 → 发关闭评论 + state: closed (not_planned)
                │
                ▼
         [log-issue-events.yml]
          记录 closed 事件到 Statsig
                │
                │ 关闭后 7 天无更新（每天 14:00 运行）
                ▼
         [lock-closed-issues.yml]
          发锁定说明评论 + 锁定 Issue (lock_reason: resolved)
          → Issue 进入只读状态

══════════════════════════════════════════════════════
                任何时候 Issue 中提到 @claude
══════════════════════════════════════════════════════

      ▼
[claude.yml]
 claude-sonnet-4-5-20250929
 读取上下文，自动响应
 （只读权限，不修改 Label）

══════════════════════════════════════════════════════
              claude-issue-triage.yml 重新触发
══════════════════════════════════════════════════════

Issue 中有新评论（非 Bot）
      │
      ▼
[claude-issue-triage.yml] 重新运行
 cancel-in-progress: true（取消旧任务）
 claude-opus-4-6 重新分类
 根据新信息更新 Label
```

---

## 9. AI 参与程度边界分析

### 9.1 完全 AI 判断（Claude 决策）

| 场景 | AI 模型 | 判断内容 |
|-----|---------|---------|
| Issue 分类 | claude-opus-4-6 | 确定 Label（bug/enhancement/invalid/needs-info 等）|
| 重复检测 | claude-sonnet-4-5-20250929 | 判断新 Issue 与已有 Issue 的相似性，提取候选重复列表 |
| @claude 响应 | claude-sonnet-4-5-20250929 | 自由回答用户的问题或执行请求 |

### 9.2 规则驱动（代码/配置决策）

| 场景 | 规则逻辑 | 代码位置 |
|-----|---------|---------|
| 生命周期超时 | 固定天数（3/7/14 天）| `issue-lifecycle.ts` |
| 标记 stale | 14 天无更新 + 无 assignee + 👍 < 10 | `sweep.ts:markStale()` |
| 关闭超时 Issue | Label 打上后超过规定天数 + 无人工评论 | `sweep.ts:closeExpired()` |
| 关闭重复 Issue | 重复评论存在 + 3 天无活动 + 无 👎 | `auto-close-duplicates.ts` |
| 锁定关闭 Issue | 关闭后 7 天无活动 | `lock-closed-issues.yml` |
| 用户反对保护 | Issue 作者 👎 = 不关闭 | `auto-close-duplicates.ts` |
| 高热度保护 | 👍 ≥ 10 = 不标记 stale | `sweep.ts`，`STALE_UPVOTE_THRESHOLD` |
| 安全检查 | allowed_non_write_users 修改 = 发警告 | `non-write-users-check.yml` |

### 9.3 AI 能力边界管控机制

**Script Caps（调用次数限制）：**
```yaml
CLAUDE_CODE_SCRIPT_CAPS: '{"edit-issue-labels.sh":2}'   # 分类最多改 2 次标签
CLAUDE_CODE_SCRIPT_CAPS: '{"comment-on-duplicates.sh":1}'  # 去重只发 1 次评论
```

**工具白名单（gh.sh）：**
- AI 只能调用 4 个只读 gh 命令
- 禁止 cross-repo 搜索
- 禁止任何写操作

**事件绑定（Shell 脚本）：**
- `edit-issue-labels.sh` 从 `GITHUB_EVENT_PATH` 读取 Issue 编号，AI 无法指定任意 Issue
- `comment-on-duplicates.sh` 限制最多 3 个候选，且必须验证 Issue 存在

---

## 10. Label 系统设计

### 10.1 生命周期 Labels（自动管理）

```
invalid      → 3 天超时关闭   （非 Claude Code 相关）
needs-repro  → 7 天超时关闭   （缺少复现步骤）
needs-info   → 7 天超时关闭   （缺少基本信息）
stale        → 14 天超时关闭  （长期无活动）
autoclose    → 14 天超时关闭  （标记即将关闭，用户可通过评论重置）
duplicate    → 立即关闭       （重复 Issue，关闭时自动打上）
```

### 10.2 分类 Labels（AI 分类）

```
bug           → Bug 报告（来自模板预设或 AI 判断）
enhancement   → 功能请求
documentation → 文档问题
model         → AI 行为问题
```

### 10.3 Label 状态转换图

```
             AI 分类
             ────────────────────────────────────────
新 Issue ──→ bug / enhancement / documentation / model  ──→ 正常处理
      │
      └──→ invalid ──→ 3 天 → 关闭
      │
      └──→ needs-repro ──→ 7 天 → 关闭
      │          └── 用户回复 → AI 重新分类 → 移除 label
      │
      └──→ needs-info ──→ 7 天 → 关闭
                 └── 用户回复 → AI 重新分类 → 移除 label

无 Label 的 Issue（14 天无更新）
      │
      └──→ stale → 14 天无活动 → 关闭
             │
             └── 用户评论 → remove-autoclose-label.yml 移除 stale → 重置

             duplicate（重复检测）
      ─────────────────────────────────────────────
新 Issue ──→ AI 检测到重复 → 发评论（3 天计时）
                │
                ├── 用户 👎 → 计时取消，不关闭
                ├── 用户评论 → 有活动，不关闭
                └── 3 天无活动 → 自动关闭 + 打 duplicate label
```

---

## 11. 关键发现总结

### 11.1 架构创新：AI + 规则混合治理

Claude Code 仓库展示了一种**成熟的混合 Issue 治理模式**：
- **AI 负责理解**：分类、重复检测这类需要"语义理解"的任务交给 Claude
- **规则负责执行**：超时、关闭、锁定等机械操作用确定性代码实现
- **用户保留否决权**：👎 保护、评论保护确保 AI 决策可被推翻

### 11.2 安全设计的深度考量

1. **最小权限原则**：每个 workflow 只申请完成任务所需的最小权限，`claude.yml` 甚至没有 `issues: write`
2. **Script Caps 防止 AI 循环**：限制脚本调用次数，防止 Claude 在标签操作上陷入无限循环
3. **工具白名单（gh.sh）**：AI 能使用的 gh 命令被严格限制，无法执行破坏性操作
4. **事件绑定防止越权**：Shell 脚本从事件 payload 读取 Issue 编号，而非接受参数，防止 AI 错误操作其他 Issue
5. **安全审计自动化**：`non-write-users-check.yml` 自动检测 `allowed_non_write_users` 的变更

### 11.3 用户体验设计

1. **"倒计时告知"原则**：每次打生命周期 Label 时立即通知用户，给出精确的剩余天数
2. **优雅的逃生通道**：用户通过 👎（反对重复）、评论（重置计时器）就能阻止自动操作
3. **高热度 Issue 保护**：10 个 👍 以上的 Issue 不会被标记 stale，社区认可度决定优先级
4. **关闭引导复开**：关闭消息总是包含"如果仍然相关，请提新 Issue"的引导

### 11.4 可观测性体系

通过 Statsig 采集三类关键事件：
- `github_issue_created`：Issue 开启（`log-issue-events.yml`）
- `github_issue_closed`：Issue 关闭（`log-issue-events.yml`）
- `github_duplicate_comment_added`：重复检测评论发出（`claude-dedupe-issues.yml`）

这为分析 Issue 处理效率、重复率、关闭原因提供数据基础。

### 11.5 工程质量亮点

1. **`issue-lifecycle.ts` 作为单一真相来源**：所有生命周期配置集中在一个文件，消除了 sweep.ts 和 lifecycle-comment.ts 之间的配置漂移风险
2. **Bun 的选择**：TypeScript 脚本无需编译即可运行，CI 环境中更快速
3. **Dry Run 模式**：`sweep.ts` 和 `backfill-duplicate-comments.ts` 都支持 `--dry-run`，运维操作可安全预览
4. **并发保护**：`claude-issue-triage.yml` 使用 `concurrency` + `cancel-in-progress` 确保同一 Issue 不会并发分类
5. **注入防护**：`log-issue-events.yml` 通过环境变量传参，避免 Shell 注入风险

### 11.6 局限性与可改进点

1. **`comment-on-duplicates.sh` 硬编码 repo**：第 11 行 `REPO="anthropics/claude-code"` 使该脚本无法复用于其他仓库
2. **重复检测的精确度依赖**：`auto-close-duplicates.ts` 通过检测评论中的文字"Found * possible duplicate"来识别 Bot 评论，字符串匹配脆弱，评论模板改动会破坏此逻辑
3. **backfill 脚本硬编码**：`backfill-duplicate-comments.ts` 中 `owner = "anthropics"`、`repo = "claude-code"` 写死，但该脚本本身是一次性工具，影响有限
4. **Statsig 上报中的 JSON 注入风险**：`log-issue-events.yml` 中 `ISSUE_TITLE` 的特殊字符处理（`sed "s/\"/\\\\\"/g"`）仅处理了双引号，换行符等仍可能破坏 JSON 结构
