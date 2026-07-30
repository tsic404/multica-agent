# multica-agent

Multica agent, skill, and squad prompts — version-controlled prompt store.

## Structure

```
multica-agent/
├── agent/          # Agent instructions (prompts)
├── skill/          # Skill content
├── squad/          # Squad instructions
└── docs/           # Architecture & deployment docs
```

## 流水线

```
① 需求分析 → ② 任务拆分 → ③ 开发+PR → ④ PR审查 → ⑤ PR QA → ⑥ Rebase合并
  Aureus       Aureus       Vulcan/Vexel   Radian      Verity       Lynx
```

## 阶段追踪

Issue 的 multi_select property 驱动。Leader 读 `issue get` 的 `properties` 字段，找第一个未勾 option = 当前阶段。

## Fork-PR 模型

Dev 推送到 tsix404 fork → 创建 PR 到 tsip404 → Review/QA 通过 `gh pr checkout` 检出 → Leader 用 `gh pr merge --rebase` 合并。

## 文档

- [架构说明](docs/architecture.md)
- [部署指南](docs/deployment.md)
