# multica-agent

Multica agent, skill, and squad prompts — version-controlled prompt store.

## 阶段追踪

每个 Property 第一个 option 为「起始」——标识类型，不推进阶段。Leader 跳过「起始」，找第一个未勾的后续 option = 当前阶段。

| Issue 类型 | Property | Options |
|-----------|----------|---------|
| Feature 父 | 需求单 | 起始, 需求分析完成, 任务拆分完成 |
| Feature 子 / Refactor | 开发单 | 起始, 开发完成, 审查通过, 测试通过 |
| Bug fix / Trivial / Doc | Bug单 | 起始, 开发完成, 审查通过, 验证通过 |
| Acceptance Test | 验收单 | 起始, 验收通过 |

## 依赖关系

文本 Property `前置依赖` 存放 CSV 格式的 TSI 列表。

## 文档

- [架构说明](docs/architecture.md)
- [部署指南](docs/deployment.md)
