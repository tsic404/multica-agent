# multica-agent

Multica agent, skill, and squad prompts — version-controlled prompt store.

## Structure

```
multica-agent/
├── agent/          # Agent instructions
├── skill/          # Skill content
├── squad/          # Squad instructions
└── docs/           # Architecture & deployment docs
```

## 自适应阶段 + DAG 依赖

Issue 的 multi_select property 驱动阶段追踪。文本 Property `前置依赖` 存放 DAG 依赖。Leader 委派前自动检查依赖是否满足。

| Issue 类型 | Property | Options |
|-----------|----------|---------|
| Feature 父 | 需求单 | 需求分析完成, 任务拆分完成 |
| Feature 子 / Refactor | 开发单 | 开发完成, 审查通过, 测试通过 |
| Bug fix / Trivial / Doc | Bug单 | 开发完成, 审查通过, 验证通过 |
| Acceptance Test | 验收单 | 验收通过 |

## Fork-PR 模型

Dev 推送到 tsix404 fork → 创建 PR 到 tsip404 → Review/QA 通过 `gh pr checkout` 检出 → Leader 用 `gh pr merge --rebase` 合并。

## 文档

- [架构说明](docs/architecture.md)
- [部署指南](docs/deployment.md)
