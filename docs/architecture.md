# 架构说明

## 三层分离

```
Autopilot（issue 间协调）
  └─ DAG 解阻塞 → 分配 Squad → Epic 管理 → 验收门禁

Squad Leader — Lynx（issue 内推进）
  └─ 读 property → 找第一个未勾 → 委派 → 读评论裁决 → 推进/回退

Agent（纯执行）
  └─ Aureus(PM) → Vulcan/Vexel(Dev) → Radian(Review) → Verity(QA) → Lynx(Merge)
```

## 阶段追踪：Property 驱动

Issue 的 multi_select property 是唯一流程真相源。不再解析 description 表格。

| Property | 适用 | Options |
|----------|------|---------|
| 需求单 | Feature 父 | 起始, 需求分析完成, 任务拆分完成 |
| 开发单 | Feature 子/Refactor | 起始, 开发完成, 审查通过, 测试通过 |
| Bug单 | Bug fix/Trivial/Doc | 起始, 开发完成, 审查通过, 验证通过 |
| 验收单 | Acceptance Test | 起始, 验收通过 |
| 前置依赖 | 所有 | text — CSV 格式 TSI 列表 |

## 流水线

```
① 需求分析 → ② 任务拆分 → ③ 开发+PR → ④ PR审查 → ⑤ PR QA → ⑥ Rebase合并
  Aureus       Aureus       Vulcan/Vexel   Radian      Verity       Lynx
```

## Fork-PR 模型

```
tsip404 (上游真相)                    tsix404 (开发 fork)
┌──────────────────────┐          ┌──────────────────────┐
│ tsip404/repo (main)  │◀──PR─── │ tsix404/repo (fork)  │
└──────────────────────┘          └──────────────────────┘
```

- Dev (Vulcan/Vexel) 推送到 tsix404 fork，创建 PR
- Review (Radian) / QA (Verity) 通过 `gh pr checkout` 检出 PR 分支
- Leader (Lynx) 用 `gh pr merge --rebase` 合并

## 设计文档

详细设计见本仓库各 agent/skill/squad 提示词，以及：

- `deployment.md` — 部署指南（Property UUID、命令模板）
- `agent/*.md` — 6 个流水线 agent 提示词
- `skill/*.md` — 8 个 skill 内容
- `squad/*.md` — Squad 指令
