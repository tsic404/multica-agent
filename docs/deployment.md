# 部署指南

## Agent 提示词部署

```bash
multica agent update <uuid> --instructions "$(cat agent/<name>.md)"
multica agent skills set <uuid> --skill-ids <id1>,<id2>,...
```

## Skill 部署

```bash
multica skill update <uuid> --content "$(cat skill/<name>.md)"
```

## Squad 部署

```bash
multica squad update <uuid> --instructions "$(cat squad/<name>.md)"
```

## 部署后

```bash
multica daemon restart
```

## 当前 Multica UUID 参考

| 实体 | Name | UUID |
|------|------|------|
| Agent | Lynx | 28546e09-e35c-431b-9fa2-b0355e2052c3 |
| Agent | Aureus | 016700c9-782f-4a43-b926-0f67e6168019 |
| Agent | Vulcan | 8101581e-c072-48bd-92d7-5d1d49d91035 |
| Agent | Vexel | 936ed413-4df8-4609-ad04-ac1bce169971 |
| Agent | Radian | 90af61ce-3e48-4ba9-a977-9d2ce5ff39d2 |
| Agent | Verity | 36355221-67bb-4ac0-a946-3ce9a53bfc27 |
| Agent | Dockerfile | 9dd6d84c-ced1-4711-8f30-4e23fcaa201b |
| Agent | Coordinator | 8a8352bd-a4e5-4fea-b011-cfacfde07710 |
| Squad | dev-team | e5f22d94-c72f-4cfd-96a0-4fc0c757aa96 |
