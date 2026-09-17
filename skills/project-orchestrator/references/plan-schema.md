<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License - see LICENSE and NOTICE. -->

# plan.json - single source of truth for the task DAG

Path: `orchestration/<project-slug>/plan.json`. The orchestrator updates it at the end of every phase.

```json
{
  "project": {
    "slug": "my-project",
    "title": "Project title",
    "goal": "One-sentence goal",
    "acceptance": ["Criterion 1", "Criterion 2"],
    "createdAt": "2026-09-03T20:40:00+08:00",
    "workspaceRoot": "<project workspace absolute path>"
  },
  "tasks": [
    {
      "id": "T01",
      "title": "Research and requirements",
      "status": "pending | running | succeeded | failed | blocked",
      "dependsOn": [],
      "tier": "standard | light",
      "tierReason": "Why this tier (one sentence)",
      "tokenBudget": {
        "estimated": 45000,
        "note": "Budget basis; if it ever exceeded the limit, record the split / read-on-demand remediation"
      },
      "briefPath": "<abs>/orchestration/<slug>/briefs/T01.md",
      "automationId": "Filled in after the subtask Automation is created",
      "modelAlias": "Actual alias used for light tier; null for standard",
      "deliverables": ["<deliverable absolute path>"],
      "acceptance": "Acceptance criteria for this subtask",
      "attempts": 0,
      "runs": [
        { "runId": "...", "startedAt": "...", "status": "succeeded", "conversationKey": "..." }
      ],
      "note": "Progress note / failure reason"
    }
  ]
}
```

## Field rules

- `id`: `T01`, `T02`... numbered in creation order, immutable afterwards.
- `dependsOn`: real artifact dependencies only. Empty array = can run in parallel immediately.
- `status`: maintained by the orchestrator; `running` means dispatched but not terminal; `blocked` means a predecessor failed.
- `tier`: `standard` omits modelAlias (follows the system default); `light` picks the model dynamically from `listModels`: prefer an alias containing `k2d8`, falling back to `defaultModelAlias` if none matches.
- `attempts`: capped at 2 (first run + 1 retry); at the cap, ask the user.
- Absolute paths only throughout.
