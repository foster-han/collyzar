# Jolli 代码生成文档流程分析

## 触发入口

有两个触发方式：

- **定时轮询**（SpacePollJobs）：每分钟检查各 source 是否有新 commit（比较最新 SHA 与存储的 cursor），有则排队 `knowledge-graph:cli-impact` job
- **GitHub Push 事件**：webhook 触发，走相同 job

---

## 完整流程概览

```
cliImpactJobHandler (KnowledgeGraphJobs.ts)
  → 获取 GitHub access token + 生成沙盒 auth token
  → runWorkflowForJob("cli-impact", ...) (workflows.ts)
      → 创建 E2B 沙盒（预装 Jolli CLI + Node.js + Git，TTL 30分钟）
      → 依次执行 6 个步骤
```

---

## 6 个关键步骤

### Step 1 — Initialize docs workspace

```bash
mkdir -p ~/docs/$GH_REPO/$GH_BRANCH
cd ~/docs/$GH_REPO/$GH_BRANCH
jolli init
jolli source add "$GH_ORG/$GH_REPO" --path "$HOME/workspace/$GH_REPO/$GH_BRANCH" --local-only
```

- 在沙盒中创建隔离的文档工作目录
- `jolli init` 初始化 Jolli workspace（生成 `.jolli/` 元数据）
- 注册代码仓库为只读 source（不同步，仅供 agent 读取代码）

---

### Step 2 — Pull docs from server

```bash
jolli sync down
# 如果指定了 TARGET_CHANGESET_ID：
jolli sync changeset checkout "$TARGET_CHANGESET_ID" --force
```

- 使用 `JOLLI_AUTH_TOKEN` + `JOLLI_SPACE` 连接后端
- 把该 space 最新文档快照拉到沙盒本地
- 如有 `TARGET_CHANGESET_ID`（增量修改场景），checkout 指定 changeset

---

### Step 3 — Analyze impact

```bash
cd ~/workspace/$GH_REPO/$GH_BRANCH
jolli impact extract ${GIT_CURSOR_SHA:+--base $GIT_CURSOR_SHA} --json > /tmp/impact-report.json
```

对比 `GIT_CURSOR_SHA`（上次处理的 commit）到 HEAD，生成 JSON 变更报告。

**impact-report.json 结构：**

```json
{
  "branch": "main",
  "base": "origin/main",
  "summary": "...",
  "commits": [
    {
      "sha": "abc1234",
      "message": "feat: add new API endpoint",
      "author": "dev",
      "summary": "",
      "hunks": [
        {
          "file": "src/api/user.ts",
          "status": "modified",
          "context": "getUser",
          "diff": "@@ -10,6 +10,8 @@\n ...",
          "queryText": ""
        }
      ]
    }
  ]
}
```

**token 爆炸根源：** 每个 hunk 的 `diff` 字段包含完整 unified diff 文本，大型仓库（如 valibot）可达 110k–130k tokens。

---

### Step 4 — Agent analysis（核心步骤）

类型：`run_prompt`，使用 Claude 模型。

**Agent 执行流程：**

```
第一轮 LLM 调用
  → 生成文本 + tool_calls（如 cat /tmp/impact-report.json）
  → 执行工具，结果追加到 history
第二轮 LLM 调用（带 tool_result）  ← 上下文溢出发生在这里
  → 继续调用工具（cat 源文件、edit_section 等）
  → ...
直到 LLM 不再发起工具调用
```

**Agent 可用工具：**

| 工具 | 用途 |
|------|------|
| `cat`, `grep`, `ls` | 读取代码和文档文件 |
| `git_diff`, `git_history`, `git_show` | 查看代码变更 |
| `create_section`, `edit_section`, `delete_section` | 编辑 markdown 文档内容 |
| `upsert_frontmatter` | 更新 YAML frontmatter |
| `write_file` | 创建新文件（用于写 changeset-metadata.json） |

**Agent 任务（来自 agentPrompt）：**

1. 读取 `/tmp/impact-report.json`，理解代码变更范围
2. 从 `~/workspace/$GH_REPO/$GH_BRANCH` 读取变更源文件（只读）
3. 更新 `~/docs/$GH_REPO/$GH_BRANCH` 下的文档
4. 为每个更新的文档添加 `attention` frontmatter，格式：`{ op: "file", source: "owner/repo", path: "<repo-relative-path>" }`
5. 写入 `/tmp/changeset-metadata.json`：`{ "message": "...", "mergePrompt": "..." }`

---

### Step 5 — Enforce attention frontmatter

类型：`run_prompt`，角色：严格的 frontmatter policy enforcer。

- 目标文件：`.jolli/changeset-checkout.json` 中列出的文件，或 workspace 中所有被修改/新增的 markdown
- 确保每个目标文件有合法的 `attention` 数组：`{ op: "file", source: "owner/repo", path: "..." }`
- 修复缺失、格式错误、重复的 attention 条目
- **绝不修改** `frontmatter.jrn`（JRN 溯源字段）

---

### Step 6 — Push updated docs

```bash
cd ~/docs/$GH_REPO/$GH_BRANCH

# 从 changeset-metadata.json 提取 message 和 mergePrompt
MESSAGE=$(node -e '...')
MERGE_PROMPT=$(node -e '...')

# 新建 changeset
jolli sync up --message "$MESSAGE" --merge-prompt "$MERGE_PROMPT"

# 或 amend 已有 changeset（当 TARGET_CHANGESET_ID 存在时）
jolli sync changeset amend "$TARGET_CHANGESET_ID" --message "$MESSAGE" --merge-prompt "$MERGE_PROMPT"
```

**输出解析（`parseCliImpactOutput`）：**

| 输出模式 | 含义 |
|----------|------|
| `"No local changes to push"` | 无文档变更 |
| `"Server changeset record ID: 12345"` | 新建 changeset |
| `"Amended changeset 12345 with 5 operations"` | 追加到已有 changeset |

---

## 端到端数据流

```
新 commit 到达
      │
      ▼
SpacePollJobs 检测到 SHA 变化
      │
      ▼
排队 knowledge-graph:cli-impact job
      │  params: { spaceId, sourceId, afterSha, cursorSha, integrationId, eventJrn }
      ▼
cliImpactJobHandler
  ├── 获取 GitHub access token
  ├── 生成沙盒 auth token（短期有效）
  └── 调用 runWorkflowForJob("cli-impact")
            │
            ▼
      E2B 沙盒（隔离环境）
        Step 1: jolli init
        Step 2: jolli sync down    ← 拉取现有文档
        Step 3: jolli impact extract  ← 分析代码变更
        Step 4: Claude agent       ← 理解变更，更新文档
        Step 5: Claude agent       ← 修复 attention frontmatter
        Step 6: jolli sync up      ← 推送文档变更
            │
            ▼
      WorkflowResult
        { hasContentChanges, changesetId, operationCount }
            │
            ▼
      后端更新 job 状态 + 推进 source cursor 到 afterSha
```

---

## 已知问题：上下文窗口溢出

**现象：** Step 4/5 的 agent 调用 `cat /tmp/impact-report.json` 后，将 tool result 追加到 history，第二轮 LLM 调用超出 200k token 限制。

**根本原因：** impact-report.json 中每个 hunk 包含完整 diff 文本，大型仓库可达 110k–130k tokens。

**修复方向：**
1. Step 3 和 Step 4 之间增加预处理步骤，生成 `impact-summary.json`
2. 仅保留 `deleted` 状态 hunk 的 diff（文件已删除，diff 是唯一的内容记录）
3. 其余 hunk（added/modified/renamed）去掉 diff，agent 直接读 checkout 中的源文件
4. Step 4/5 的 prompt 改为读 `impact-summary.json`

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| `backend/src/jobs/SpacePollJobs.ts` | 定时轮询，检测新 commit，排队 job |
| `backend/src/jobs/KnowledgeGraphJobs.ts` | job handler，准备参数，调用 workflow |
| `tools/jolliagent/src/workflows.ts` | workflow 定义与执行（6 步骤，`runWorkflowForJob`） |
| `tools/jolliagent/src/agents/Agent.ts` | Claude agent，管理 LLM 对话和工具调用循环 |
| `tools/jolliagent/src/sandbox/E2BSandboxFactory.ts` | E2B 沙盒创建 |
| `cli/src/client/commands/impact.ts` | `jolli impact extract` 命令实现 |
