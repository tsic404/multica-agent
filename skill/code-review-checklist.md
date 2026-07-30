# 代码审查检查清单 v2.0

## 触发条件

Radian 被委派代码审查任务时加载。按以下流水线执行：**前置分析 → 逐维检查 → 输出 verdict → 自检**。

---

## 0. 前置分析（必须第一步完成）

### 0.1 获取 diff 元数据

```bash
git fetch origin "$REMOTE_BRANCH" && git checkout "$REMOTE_BRANCH"
DIFF_STAT=$(git diff --stat origin/main..."$REMOTE_BRANCH")
DIFF_LINES=$(echo "$DIFF_STAT" | tail -1 | grep -oP '\d+')
DIFF_FILES=$(git diff --name-only origin/main..."$REMOTE_BRANCH" | wc -l)
```

### 0.2 确定规模（L 级别）—— 决定审查策略

| L | 行数 | 策略 |
|----|------|------|
| L0 | 1–50 | 逐行细审 |
| L1 | 50–200 | 逐文件审，关注逻辑和边界 |
| L2 | 200–500 | 优先核心逻辑文件，配置/格式变更快速扫 |
| L3 | 500–1000 | 按模块分组，关注接口边界 |
| L4 | 1000+ | **直接 REQUEST CHANGES**。不逐行审。 |

### 0.3 确定改动类型 —— 决定审查重心

```bash
git diff --name-only origin/main..."$REMOTE_BRANCH"
```

根据文件列表判断：

| 类型 | 特征 | 重心放在 |
|------|------|---------|
| 新功能 | 新增文件/函数/API | 架构、边界、API 设计、测试 |
| Bug 修复 | 改动集中在少数函数 | 根因正确性、回归风险 |
| 重构 | 大量移动/重命名 | 行为等价性、夹带私货 |
| 配置 | yaml/json/env/Dockerfile | 安全（secret）、默认值 |
| 依赖 | go.mod/Cargo.toml/package.json 等 | 版本原因、breaking、CVE |
| 迁移 | SQL/schema 变更 | 向后兼容、回滚、数据丢失 |

### 0.4 审查优先级排序

1. 业务逻辑（src/services/、pkg/、internal/）
2. API 定义（proto、openapi、路由）
3. 数据访问（models、repos、migrations）
4. 工具函数（utils、helpers）
5. 测试文件
6. 配置和文档

---

## 1. 逐维检查

对每一维：逐条过 → 有问题标 `[文件:行号]` → 判断阻塞/建议。

### 1.1 代码质量

- [ ] 逻辑正确：条件、循环、边界（空数组/nil/零值/极值）
- [ ] 无 unreachable code、注释掉的代码、dead code
- [ ] 错误处理：所有可能失败的操作有处理，不吞异常
- [ ] 空 catch/except 必须有注释说明原因，否则 → **阻塞**
- [ ] 资源释放：defer / context manager / `.close()`
- [ ] 函数单一职责，长度不超标（Go ≤80，Python ≤60，TS ≤50 行）
- [ ] 命名准确表达意图，无含糊缩写
- [ ] 魔法数字 → 命名常量

### 1.2 架构一致性

- [ ] 新代码放在正确的模块/目录
- [ ] 无破坏现有接口的变更（API 签名、DB schema、配置格式）
- [ ] 无循环依赖
- [ ] 一个接口只有一个实现 → 过度设计，标记建议
- [ ] 未重复造轮子（检查项目中是否已有类似工具函数）

### 1.3 安全风险（**阻塞性**：任何一项不通过 → REQUEST CHANGES）

- [ ] 无硬编码密钥/密码/Token/私钥
- [ ] 无硬编码内部 IP/域名/端口
- [ ] 用户输入已校验和转义（XSS、路径遍历）
- [ ] SQL/命令使用参数化，无字符串拼接
- [ ] 敏感操作有鉴权检查
- [ ] 日志不输出敏感信息（密码/手机号/身份证/Token）
- [ ] 文件上传有类型和大小限制
- [ ] 无 `os.system()` / `subprocess(shell=True)` 拼接用户输入

### 1.4 编码规范

- [ ] 缩进、命名、行宽遵循项目约定（≤120 字符）
- [ ] 导入分组：stdlib → 第三方 → 项目内
- [ ] 注释为英文或与项目一致

### 1.5 性能（按改动类型选查：新功能/重构 → 全查；配置/迁移 → 跳过）

- [ ] 无循环内数据库查询（N+1）
- [ ] 无循环内 HTTP 调用
- [ ] 大数据使用流式/分批，不全量加载
- [ ] 无不必要的深拷贝或大对象复制

### 1.6 并发安全（Go/Rust/Python async 项目 → 必查）

- [ ] 共享可变状态有锁保护
- [ ] 无 goroutine/task 泄露（所有启动有退出机制）
- [ ] channel 正确关闭
- [ ] 无数据竞争
- [ ] 锁持有时间合理，不在持锁时做 I/O

### 1.7 API 兼容性（有 API 变更 → 必查）

- [ ] 新增字段 optional，删除/重命名标记 breaking
- [ ] 响应格式不破坏现有客户端
- [ ] 错误码和 HTTP 状态码正确

### 1.8 测试质量

- [ ] 新增功能有测试
- [ ] 覆盖正常路径 + 至少一个异常路径
- [ ] 覆盖关键边界条件
- [ ] 测试命名清晰描述场景
- [ ] 无 flaky 测试（依赖时间/随机数/外部服务/全局状态）
- [ ] Mock 不过度（测了 mock 不是测了代码）

### 1.9 语言专项

常见检查点见 `references/language-checks.md`。反模式速查见 `references/anti-patterns.md`。

**执行规则**：遍历 diff 中出现的文件类型，针对每种语言过一遍 references 中的检查清单。

---

## 2. 输出 Verdict

**EXACTLY ONE comment。必须包含 verdict 关键词 + `流程已更新`。**

## 提交审查结果

审查完成后，**双通道提交**：

### 1. GitHub PR Review
```bash
# APPROVED
gh pr review "$PR_URL" --approve --body "审查摘要..."

# REQUEST CHANGES  
gh pr review "$PR_URL" --request-changes --body "🔴 阻塞: ..."

# REVIEW BLOCKED
gh pr review "$PR_URL" --comment --body "阻塞原因..."
```

### 2. Multica Issue 评论
发一条评论含粗体裁决（Lynx 解析用）。

---

### APPROVED

```
**APPROVED**

**审查摘要**
- 规模：L[N]（[N] 行，[N] 文件）
- 类型：[新功能/Bug修复/重构/配置/依赖/迁移]
- 覆盖：代码质量 ✓ | 安全 ✓ | 架构 ✓ | 规范 ✓ | [其他] ✓
- 分支：`multica/[IDENTIFIER]`
- 提交：`[short SHA]`

[改动内容和通过原因，1–2 句]

流程已更新。
```

## 提交审查结果

审查完成后，**双通道提交**：

### 1. GitHub PR Review
```bash
# APPROVED
gh pr review "$PR_URL" --approve --body "审查摘要..."

# REQUEST CHANGES  
gh pr review "$PR_URL" --request-changes --body "🔴 阻塞: ..."

# REVIEW BLOCKED
gh pr review "$PR_URL" --comment --body "阻塞原因..."
```

### 2. Multica Issue 评论
发一条评论含粗体裁决（Lynx 解析用）。

---

### APPROVED（附建议）

```
**APPROVED**

**审查摘要**
- 规模：L[N] · 类型：[类型] · 分支：`multica/[IDENTIFIER]`

**非阻塞建议**
- `[文件:行号]` — [建议]
- `[文件:行号]` — [建议]

流程已更新。
```

### REQUEST CHANGES

```
**REQUEST CHANGES**

**审查摘要**
- 规模：L[N] · 类型：[类型] · 分支：`multica/[IDENTIFIER]`
- 通过：代码质量 [?] | 安全 [?] | 架构 [?] | 规范 [?]

**阻塞问题**
1. `[文件:行号]` — [维度] — [问题]
   → 修复：[具体做法]
2. `[文件:行号]` — [维度] — [问题]
   → 修复：[具体做法]

请修复后重新提交。

流程已更新。
```

### REVIEW BLOCKED

```
**REVIEW BLOCKED**

**阻塞原因**（取一）：
- diff 为空，无代码变更
- 规模 L4（[N] 行），请拆分为多个 MR
- 分支 `multica/[IDENTIFIER]` 不存在或无法检出
- 构建失败，无法验证代码
- issue 范围不清晰（无验收标准）

流程已更新。
```

---

## 3. 输出前自检（Verification）

提交 comment 前逐条确认：

- [ ] 只有 ONE comment
- [ ] verdict 关键词加粗：`**APPROVED**` / `**REQUEST CHANGES**` / `**REVIEW BLOCKED**`
- [ ] 包含 `流程已更新`
- [ ] 每个阻塞问题都附了 `[文件:行号]` + 修复建议
- [ ] 安全维度 8 条全部检查过
- [ ] 没有写代码、改文件、跑测试、assign issue
- [ ] 没有内部独白或进度更新

---

## Pitfalls（Radian 已知翻车模式）

| # | 翻车 | 原因 | 预防 |
|----|------|------|------|
| 1 | 空 diff 也输出 APPROVED | git checkout 失败但未检查退出码 | Startup 脚本 `set -euo pipefail`，diff 为空先判断 |
| 2 | 审了一半就发 comment | 未完成全部维度检查 | 过完 1.1–1.9 全部后再输出 |
| 3 | 安全维度漏审 | 被大量代码质量发现淹没 | 安全是阻塞性，先审安全再审质量 |
| 4 | 不标文件:行号 | 笼统说"某处有问题" | 输出模板强制要求 `[文件:行号]` |
| 5 | 发了两个 comment | 先发了 APPROVED 又补一个 | 禁止：输出前自检确认只发一次 |
| 6 | 分不清阻塞和建议 | 把所有发现都当阻塞 | 安全/逻辑错误 = 阻塞。命名/注释/风格 = 建议 |
| 7 | 没有 issue 上下文 | 不知道改动的目的是什么 | startup 脚本 `multica issue get` 先读 issue 标题和描述 |
| 8 | 超 1000 行还逐行审 | 超大 diff 审不完中途截断 | L4 直接 REQUEST CHANGES |

---

## References

- `references/language-checks.md` — Go / Python / TS&JS / Rust / Dockerfile / YAML / SQL 专项检查清单
- `references/anti-patterns.md` — 常见反模式速查表（13 条）

