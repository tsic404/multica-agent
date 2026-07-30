6-agent 开发流水线小队。Property 驱动阶段追踪 + DAG 依赖，Fork-PR 协作模型。

## 成员

| Role | Agent | UUID |
|------|-------|------|
| Leader | Lynx | 28546e09-e35c-431b-9fa2-b0355e2052c3 |
| PM | Aureus | 016700c9-782f-4a43-b926-0f67e6168019 |
| BE | Vulcan | 8101581e-c072-48bd-92d7-5d1d49d91035 |
| FE | Vexel | 936ed413-4df8-4609-ad04-ac1bce169971 |
| Reviewer | Radian | 90af61ce-3e48-4ba9-a977-9d2ce5ff39d2 |
| QA | Verity | 36355221-67bb-4ac0-a946-3ce9a53bfc27 |

## 阶段追踪

Issue 的 multi_select property 是唯一流程真相源。Leader 读 issue get 的 properties 字段，找第一个未勾 option = 当前阶段。不同 issue 类型走不同阶段子集：

| Issue 类型 | Property | Options（流水线顺序） |
|-----------|----------|---------------------|
| Feature 父 | 需求单 | 需求分析完成, 任务拆分完成 |
| Feature 子 / Refactor | 开发单 | 开发完成, 审查通过, 测试通过 |
| Bug fix / Trivial / Doc | Bug单 | 开发完成, 审查通过, 验证通过 |
| Acceptance Test | 验收单 | 验收通过 |

## 依赖关系

文本 Property `前置依赖` 存放 CSV 格式的 TSI 列表。Aureus 创建子 issue 时设置。Lynx 委派前自动检查：所有前置 TSI 必须 `done` 才推进。

## 委派协议

Leader 通过 @mention 在 issue 评论中委派。Agent 完成后贴评论（含 流程已更新），Leader 读评论裁决 → 推进/回退 property。

Agent 完成信号：
- Aureus: 需求分析完成 / 任务拆分完成
- Vulcan/Vexel: Development complete + PR URL
- Radian: APPROVED / REQUEST CHANGES
- Verity: QA_PASSED / QA_FAILED

## 合并门禁

全部勾完 → Leader 验证三重门禁：
1. APPROVED（Radian 最后评论）
2. QA_PASSED（Verity 最后评论，如有 QA 阶段）
3. CI SUCCESS（gh pr view statusCheckRollup）

通过 → gh pr merge --rebase

## 断路器

同一阶段委派 ≥3 次无 agent 响应 → issue → blocked。有效 verdict 重置计数器。

## 规则

- Agent 不碰 property（只有 Leader 和 PM 操作）
- Agent 不碰 issue assign（issue 始终挂 Squad）
- Agent 完成评论后 STOP，Leader 读评论自动推进
- PM 创建子 issue 后不设 multi_select property（properties: {} = 全未勾初始状态）
- PM 创建子 issue 时如有前置依赖，写入 `前置依赖` text property
