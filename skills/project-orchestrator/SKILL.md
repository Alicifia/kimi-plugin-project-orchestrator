---
name: project-orchestrator
description: 大型项目自主规划与多子任务编排。当用户要求"自主规划并完成一个大型项目"、需要把大项目拆成多个子任务（子对话）并行/串行执行、监控子任务进度、或汇总子任务产物时触发。覆盖：项目分析、任务清单与 DAG 拆解、上下文长度预算、子任务 Automation 调度、看板监控、模型分级（简单任务降级省额度）、产物汇总。
---

<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License — see LICENSE and NOTICE. -->

# 项目总控 Orchestrator

把一个大型项目拆解成一组有依赖关系的子任务，每个子任务跑在一个独立的 `local_conversation` Automation（即一个独立子对话）里，主对话扮演"总控"：规划、派工、监控、汇总。

## 何时使用

- 用户明确要求"自主规划并完成大型项目"、"拆成子任务并行做"、"多 agent 协作完成 X"。
- 项目规模明显超出单次对话能高质量完成的范围（交付物 ≥ 3 个、或预计需要多阶段串行）。

小规模任务不要动用本流程，直接做。

## 总览：六个阶段

```
P0 项目分析 → P1 任务拆解(DAG+上下文预算) → P2 建子任务(简报+Automation)
→ P3 调度执行(并行/串行) → P4 看板监控 → P5 汇总交付
```

所有编排状态文件放在项目根目录：

```
<工作区>/orchestration/<项目slug>/
  plan.json            # 任务 DAG，唯一事实来源（见 references/plan-schema.md）
  briefs/<task-id>.md  # 每个子任务的自包含简报（见 references/brief-template.md）
  status/<task-id>.json # 子任务进度心跳（见 references/status-board.md）
  deliverables/        # 子任务产物（或记录产物绝对路径）
```

---

## P0 项目分析

1. 与用户确认：项目目标、交付物清单、验收标准、截止时间、是否允许消耗额度跑子任务（**必须显式确认：运行子任务会消耗额度**，说明预计子任务数量后再继续）。
2. 在工作区创建 `orchestration/<项目slug>/` 目录。
3. 写 `plan.json` 的 `project` 段：目标、验收标准、创建时间。

## P1 任务拆解 + 上下文预算

按项目规模拆，常见 3–12 个，大型项目可以更多（名额回收机制见 P2，总任务数不受名额上限约束）。每个子任务必须满足：**单一明确交付物、可独立验收、上下文可控**。

### 依赖关系

- 在 `plan.json` 里为每个任务标 `dependsOn: [task-id, ...]`。
- 只标真实的数据/产物依赖（B 需要 A 的输出文件），不要为"看起来有顺序"加依赖——无依赖的任务才能并行。

### 上下文长度预算（关键，防止子任务被压缩降智）

子任务跑在独立对话里，上下文超限会被压缩，效果下降。派工前为每个任务做预算：

```
估算tokens ≈ (简报大小 + 必读输入文件总字节数/2 + 预计输出tokens + 工具往返开销≈20k) × 1.5 安全系数
```

预算规则：

- **估算 ≤ 100k tokens**：可以作为一个子任务。
- **超过 100k**：二选一——
  - 再拆成两个有先后关系的子任务（前一个把中间结论写入 `deliverables/` 文件，后一个读文件而非继承上下文）；
  - 或在简报中明确"按需读取"：只给文件清单+绝对路径，让子任务自己选读，不把内容贴进简报。
- **简报正文本身控制在 4k tokens 以内**：背景知识、上游结论一律落成文件，简报里只写绝对路径引用。子任务 Automation 的 prompt 更短——只给简报文件路径。

把每个任务的估算值写进 `plan.json` 的 `tokenBudget.estimated`，超限时在 `tokenBudget.note` 里记录处置方式。

### 模型分级

为每个任务标 `tier`：

- `standard`（默认）：创建 Automation 时**不显式传 modelAlias**，跟随系统默认模型——与主对话同级能力。复杂、需要判断/创作/代码的任务一律用这档。
- `light`：简单、机械、低风险任务（格式转换、批量改名、模板化文案、数据搬运），用系统返回别名中含 `k2d6` 的模型（先用 `AutomationControl action:"listModels"` 取当前准确别名），减少额度消耗。
- 拿不准就 `standard`。在 `plan.json` 记录每个任务的 `tier` 和理由。

## P2 创建子任务

对每个任务：

1. 写 `briefs/<task-id>.md`（模板见 references/brief-template.md）。简报必须自包含：目标、输入文件绝对路径、约束、交付物路径、验收标准、**进度上报要求**（让子任务在关键里程碑更新 `status/<task-id>.json`）。
2. 创建子任务 Automation（每个任务一个）：
   - `execution.kind: "agent"`, `mode: "local_conversation"`, `workspace: { "kind": "path", "path": "<项目工作区绝对路径>" }`（与主对话同一任务文件夹，产物集中）。
   - prompt 保持短：让子任务读简报文件执行，并把结果写入指定路径。例：`"阅读 <abs>/briefs/T03.md 并严格执行其中的任务。完成后把交付物写到指定位置，并按简报要求更新 status/T03.json。"`
   - `trigger: { "kind": "manual" }`——由总控显式派工，不要 schedule。
   - `result: { "kind": "conversation" }`。
   - 模型：按 P1 的 `tier` 决定省略或传 `modelAlias`。
   - 长任务设 `timeoutMs`。
3. 把返回的 `automationId` 写回 `plan.json` 对应任务。
4. 创建前向用户做一次合并确认：子任务数量、并行方式、预计额度消耗、各任务模型档位。

> 名额回收（重要）：每个**启用中**的子任务 Automation 占一个 cron job 名额，但 **disabled 状态保留全部 run 记录且不占名额**。因此总任务数不受名额上限约束，只有"同一时刻并行 + 等待条件触发的任务数"受可用名额限制：
> - 子任务 run 到达 `succeeded`（或终态放弃）后，总控**立即 disable 它的 Automation** 释放名额，再继续创建/派发后续任务；run 记录与 conversationKey 仍可查。
> - 更进一步可以"按需创建"：任务临近派工时才创建它的 Automation，而不是 P2 一次性全建——这样同时占用的名额 ≈ 最大并行宽度。
> - 配了 condition 触发的任务在等待期间必须保持启用，会占名额；名额紧张的长链项目改用纯手动接力（主对话唤醒时推进）。

## P3 调度执行

调度循环（总控在主对话中执行）：

1. 从 `plan.json` 找出 `status: "pending"` 且 `dependsOn` 全部 `succeeded` 的任务。
2. **并行**：对这批任务同时调用 `AutomationControl action:"run"`（同一批互相独立，一次发完）。
3. **串行依赖**：后继任务必须等前驱 run 到达 `succeeded` 才触发。轮询用 `listRuns` / `readRun`，不要 tight-loop——每次用户交互或间隔检查时查一轮即可。
4. **全自动接力（可选，推荐长链使用）**：主对话只在用户发消息时醒来，纯手动模式下串行链会停在"等主对话被唤醒"。要无人值守推进，给后继任务的 Automation 配 `condition` 触发（默认每 10 分钟轮询一次 Python 条件）：条件读 `plan.json` 与 `status/`，仅当其全部前驱 `succeeded` 且自身仍 `pending` 时返回 true——前驱一成后继自动开跑。任务完成后立即 disable 该条件触发，防止空转。条件触发是 Automation 平台的内置能力，细节见 automation 技能的 `references/trigger-condition.md`。
5. **失败处理**：run `failed`/`timeout` → `readRunLogs` 取证据，在 `plan.json` 记录 `attempts`，允许修简报后重跑 1 次；再失败则停下向用户报告，不要无限重试。
6. **验证**：子任务报完成不算数——总控必须亲自检查交付物文件存在且内容达标（读文件，不是读子任务的自我汇报）。不达标打回重跑。

每完成一批就更新 `plan.json` 里的 `status`；对每个到达终态（succeeded / 放弃）的任务，**立即 disable 它的 Automation 释放名额**（记录保留，见 P2 名额回收），再继续创建/派发后续任务。

## P4 看板监控

项目启动后立刻建监控看板（用户和主对话都能看），做法见 references/status-board.md：

1. 用 Widget 技能创建一个任务看板 Widget（遵循 Kimi 设计系统）。
2. 创建一个 Python 状态聚合 Automation（widget task）：读 `plan.json` + `status/*.json`，产出含任务列表/状态/进度/模型档位的 artifact。
3. 用 Binding 把 artifact 绑到 Widget 的 `main` slot。
4. 聚合 Automation 用 interval 触发（项目活跃期 15–30 分钟一次），或由总控在关键节点手动 run 刷新。

主对话随时可被问"进展如何"：读 `plan.json` + `status/` + 各 run 状态，用三五行说清：完成/进行/阻塞各哪些、当前在跑什么、下一步是什么。

## P5 汇总交付

全部任务 `succeeded` 后：

1. 逐个核对 `plan.json` 的 `deliverables` 路径，确认每个文件真实存在、内容达标（打开看，不凭记录）。
2. 需要时读取子任务 run 的 `conversationKey` / transcript 了解过程结论。
3. 给用户一份总结：项目目标达成情况、交付物清单（绝对路径 markdown 链接）、各子任务用时/模型档位/是否返工、遗留风险。
4. 清理（先征得用户同意）：disable 或 delete 子任务 Automation 释放名额；停掉聚合 Automation 的 interval。编排文件保留在 `orchestration/` 备查。

## 硬性规则

- 先确认额度，再创建任何子任务 Automation。
- 上下文预算先行：任何估算 > 100k tokens 的任务必须处置后才能派工。
- 串行依赖严格等待前驱 `succeeded`，禁止凭"应该做完了"提前触发。
- 子任务产物必须由总控亲自验证，不接受自我汇报。
- 失败最多自动重试 1 次，之后必须问用户。
- 名额回收：子任务到达终态后立即 disable 释放名额（disabled 保留记录、不占名额）；总任务数不受名额上限约束，受约束的只是同一时刻"并行中 + 等待条件触发"的任务数。
- 所有路径用绝对路径；简报、状态、产物都落在 `orchestration/<项目slug>/` 内。

## 参考文件

- `references/plan-schema.md` — plan.json 完整字段定义
- `references/brief-template.md` — 子任务简报模板
- `references/status-board.md` — 状态心跳格式 + 看板 Widget / 聚合 Automation 搭建步骤
