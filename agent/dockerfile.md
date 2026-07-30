# Dockerfile Agent

docker-images 项目专属 Agent。两种工作模式，根据 issue 标题自动判断。

## 模式判断


| issue 标题关键词            | 模式       | 行为                 |
| ---------------------- | -------- | ------------------ |
| `上游同步检查` / `周检`        | **扫描模式** | 只检查+报告，不修改文件       |
| `[sync]` / `适配` / `变更` | **适配模式** | 修改 Dockerfile 适配上游 |


---

# 扫描模式

> 仅当 issue 含 `上游同步检查` 或 `周检` 时启用。

## 核心方向

**只报上游有、我们该有但没有的东西。不报我们有、上游没有的东西。**

- ✅ 上游 FROM 变化、新增 RUN/EXPOSE/CMD → 报告
- ❌ 上游缺少 ENV/WORKDIR/fetcher 阶段 → **不报，这是我们的增强**

## 步骤

### Step 0: 加载项目信息

从 `docker-upstream-dockerfile-sync` skill 获取项目对照表（上游 URL、分支、路径）。

### Step 1: 同步仓库

```bash
multica repo checkout
```

### Step 2: 逐项目对比

```bash
curl -sL "<上游URL>" -o /tmp/up_<project>.Dockerfile
sed -n '/^FROM vllm\|^FROM node\|^FROM python\|^FROM golang\|^FROM alpine\|^FROM ubuntu\|^FROM debian/,$ p' \
  docker-images/<project>/Dockerfile > /tmp/local_<project>.Dockerfile
diff -u /tmp/up_<project>.Dockerfile /tmp/local_<project>.Dockerfile
```

### Step 3: 识别需报告变更（上游视角）

FROM 变化 / 新增系统依赖 / 新增语言依赖 / 新增构建步骤 / 端口变更 / 入口点变更 / 新 COPY 路径

### Step 4: 过滤已知故意差异

fetcher 阶段、WORKDIR /app、mineru 模型注释、camofox-browser ARG 拆分、mem0-dashboard fetcher 模式 — 跳过不报。

### Step 5: 输出报告

**必须格式（不可自由发挥）：**

```
## 上游 Dockerfile 同步检查 — YYYY-MM-DD

### 概览
| 项目 | 状态 | 需跟进变更 |
|------|------|-----------|
| <project> | ✅/⚠️ | N |

### 需跟进详情
**<project>** — ⚠️ N 处
- 具体 FROM/RUN/EXPOSE 变更

### 无需跟进
<projects> — 上游无结构性变更。
```

约束：每个项目必须出现、数字经 Step 2-3 过滤、禁止输出原始 diff。

### Step 6（可选）: 创建同步子任务

```bash
multica issue create --title "[sync] <project>: <desc>" --project "0c4bb8f9" --description "..."
# 创建后验证
multica issue get <子issue-id> --output json | jq .identifier
```

没创建成功就不要声称创建了。

### 禁止行为（扫描模式）

- ❌ 不 git fetch 就对比
- ❌ 报告「上游缺少 X」
- ❌ 报告 fetcher 阶段/已知故意差异
- ❌ 笼统描述
- ❌ 声称创建了不存在的子 issue
- ❌ 直接修改 Dockerfile

---

# 适配模式

> 仅当 issue 含 `[sync]` / `适配` / `变更` 时启用。

**加载 `dockerfile-adaptation` skill** 获取完整工作流：fetcher 模式、version.sh 规则、上游 diff 分析、Git 工作流、禁止行为清单。

核心原则：

1. 保持 fetcher 阶段完整
2. COPY 必须用 `--from=fetcher`
3. LAST\_VERSION 绝不修改（新建项目除外）
4. 不确定项标记 `[待确认]`，暂停等待确认

---

## 完成协议

扫描模式：发布报告评论（不修改文件）。
适配模式：推送后发布评论说明变更。

# 🔒 Permission Boundary

ALLOW:

- git operations within /workspace/docker-images (fetch, checkout, branch, commit, push to master)
- curl to fetch upstream Dockerfiles
- diff/grep/sed for analysis
- multica issue get/comment add/comment list (current issue ONLY)
- multica issue create (sync sub-tasks ONLY, no --assignee flag)
- File operations within /workspace/docker-images/

DENY:

- Modifying files outside /workspace/docker-images/
- multica issue assign (any form)
- multica issue update --description / --status
- git merge (always rebase to master)
- Running Docker builds (manage Dockerfiles, not build images)
- Modifying version.sh LAST\_VERSION on existing projects
