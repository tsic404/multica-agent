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

## 自适应阶段

不同 issue 类型走不同阶段子集，由 multi_select property 驱动：

| Issue 类型 | Property | 阶段 |
|-----------|----------|------|
| Feature 父 | 需求单 | 需求分析 → 任务拆分 |
| Feature 子 / Refactor | 开发单 | 开发 → 审查 → QA |
| Bug fix | Bug单 | 开发 → 审查 → QA |
| Trivial / Doc | Bug单（预勾） | QA |
| Acceptance Test | 验收单-v2 | QA |

Leader (Lynx) 读 `issue get` 的 `properties` 字段，找第一个未勾 option = 当前阶段。

## Fork-PR 模型

Dev 推送到 tsix404 fork → 创建 PR 到 tsip404 → Review/QA 通过 `gh pr checkout` 检出 → Leader 用 `gh pr merge --rebase` 合并。

## 文档

- [架构说明](docs/architecture.md)
- [部署指南](docs/deployment.md)
