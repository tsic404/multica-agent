# 新项目开发 Setup

新项目标准初始化流程。创建 tsip404 主仓库 + tsix404 fork + Multica 项目。

## 触发条件
- 用户要求创建新项目
- 新仓库需要接入 Multica 流水线

---

## Step 1: tsip404 创建主仓库

```bash
# 切换到 tsip404 身份
gh auth switch --user tsip404

# 创建仓库（公开，main 分支）
gh repo create tsip404/<repo-name> --public --description "<项目描述>"
```

## Step 2: 设置分支保护规则

```bash
gh api repos/tsip404/<repo-name>/branches/main/protection \
  -X PUT --input - <<'JSON'
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": false,
  "lock_branch": false,
  "allow_fork_syncing": true
}
JSON
```

保护效果：
- ❌ 禁止 force push
- ❌ 禁止删除分支  
- ✅ 仅 tsip404 可 push（个人仓库无协作者）
- ✅ PR 合并由 tsip404（Lynx）执行

## Step 3: tsix404 fork 开发仓库

```bash
# 切换到 tsix404 身份
gh auth switch --user tsix404

# Fork（不 clone 到本地）
gh repo fork tsip404/<repo-name> --clone=false --remote=false
```

## Step 4: Multica 创建项目 + 挂载仓库

```bash
# 切换回 tsip404（Multica workspace 默认身份）
gh auth switch --user tsip404

multica project create \
  --title "<项目名称>" \
  --description "<项目描述>" \
  --icon "<emoji>" \
  --repo "https://github.com/tsip404/<repo-name>" \
  --output json
```

`--repo` 自动创建 github_repo 类型 resource 并关联到项目。

## Step 5: 验证

```bash
# 主仓库存在
gh repo view tsip404/<repo-name> --json name,defaultBranchRef

# Fork 存在
gh repo view tsix404/<repo-name> --json name,parent

# Multica 项目
multica project list --output json | jq '.[] | select(.title == "<项目名称>") | {id, title, repo_name}'

# 分支保护
gh api repos/tsip404/<repo-name>/branches/main/protection --jq '{force_push: .allow_force_pushes.enabled, deletions: .allow_deletions.enabled}'
```

---

## 常见问题

### Q: 仓库已存在？
跳过 Step 1，从 Step 2 开始。

### Q: Fork 已存在？
跳过 Step 3。如需同步：`gh repo sync tsix404/<repo-name> --source tsip404/<repo-name>`

### Q: 需要私有仓库？
`gh repo create` 加 `--private` 标志。fork 自动继承可见性。
