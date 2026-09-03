<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License — see LICENSE and NOTICE. -->

# plan.json — 任务 DAG 唯一事实来源

路径：`orchestration/<项目slug>/plan.json`。总控在每个阶段结束后更新它。

```json
{
  "project": {
    "slug": "my-project",
    "title": "项目标题",
    "goal": "一句话目标",
    "acceptance": ["验收标准1", "验收标准2"],
    "createdAt": "2026-09-03T20:40:00+08:00",
    "workspaceRoot": "<项目工作区绝对路径>"
  },
  "tasks": [
    {
      "id": "T01",
      "title": "调研与需求整理",
      "status": "pending | running | succeeded | failed | blocked",
      "dependsOn": [],
      "tier": "standard | light",
      "tierReason": "为什么是这一档（一句话）",
      "tokenBudget": {
        "estimated": 45000,
        "note": "预算依据；若曾超限，记录拆分/按需读取的处置方式"
      },
      "briefPath": "<abs>/orchestration/<slug>/briefs/T01.md",
      "automationId": "创建子任务 Automation 后回填",
      "modelAlias": "light 档时回填实际使用的别名；standard 档留 null",
      "deliverables": ["<产物绝对路径>"],
      "acceptance": "本子任务的验收标准",
      "attempts": 0,
      "runs": [
        { "runId": "...", "startedAt": "...", "status": "succeeded", "conversationKey": "..." }
      ],
      "note": "进展备注/失败原因"
    }
  ]
}
```

## 字段规则

- `id`：`T01`、`T02`…… 按创建顺序编号，创建后不变。
- `dependsOn`：只列真实产物依赖。空数组 = 可立即并行。
- `status`：由总控维护；`running` 表示已派工未到终态；`blocked` 表示前驱失败。
- `tier`：`standard` 不显式传 modelAlias（跟随系统默认）；`light` 用 `listModels` 返回的含 `k2d6` 的别名。
- `attempts`：上限 2（首跑 + 1 次重试），到顶必须问用户。
- 全文只用绝对路径。
