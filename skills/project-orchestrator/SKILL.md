---
name: project-orchestrator
description: Autonomous planning and multi-subtask orchestration for large projects. Trigger when the user asks to "autonomously plan and complete a large project", needs a big project split into subtasks (sub-conversations) executed in parallel/serial, wants subtask progress monitoring, or needs deliverable aggregation. Covers: project analysis, task-list and DAG decomposition, context-length budgeting, subtask Automation dispatch, kanban monitoring, model tiering (downgrade simple tasks to save credits), and deliverable aggregation.
---

<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License - see LICENSE and NOTICE. -->

# Project Orchestrator

Decompose a large project into a set of dependency-linked subtasks. Each subtask runs in its own `local_conversation` Automation (an independent sub-conversation), while the main conversation acts as the orchestrator: plan, dispatch, monitor, aggregate.

## When to use

- The user explicitly asks to "autonomously plan and complete a large project", "split this into parallel subtasks", or "have multiple agents collaborate on X".
- The project clearly exceeds what a single conversation can deliver at high quality (3+ deliverables, or multiple sequential stages expected).

Do not use this workflow for small tasks - just do them directly.

## Overview: six phases

```
P0 Project analysis -> P1 Task decomposition (DAG + context budget) -> P2 Create subtasks (briefs + Automations)
-> P4 Board setup (completed BEFORE dispatch) -> P3 Dispatch & execution (parallel/serial) -> P5 Aggregation & delivery
```

All orchestration state files live under the project root:

```
<workspace>/orchestration/<project-slug>/
  plan.json             # Task DAG, single source of truth (see references/plan-schema.md)
  briefs/<task-id>.md   # Self-contained brief per subtask (see references/brief-template.md)
  status/<task-id>.json # Subtask progress heartbeat (see references/status-board.md)
  deliverables/         # Subtask outputs (or their absolute paths)
```

---

## P0 Project analysis

1. Confirm with the user: project goal, deliverables list, acceptance criteria, deadline, and whether credit consumption for subtasks is allowed (**explicit confirmation is required: running subtasks consumes credits** - state the expected subtask count before proceeding).
2. Create `orchestration/<project-slug>/` in the workspace.
3. Write the `project` section of `plan.json`: goal, acceptance criteria, creation time.

## P1 Task decomposition + context budget

Decompose by project size - typically 3-12 subtasks, more for very large projects (slot recycling in P2 means total task count is not capped). Every subtask must satisfy: **one clear deliverable, independently verifiable, context-bounded**.

### Dependencies

- Mark `dependsOn: [task-id, ...]` for each task in `plan.json`.
- Only record real data/artifact dependencies (B needs A's output file). Never add dependencies for superficial ordering - only dependency-free tasks can run in parallel.

### Context-length budgeting (critical - prevents subtask quality loss from context compression)

Each subtask runs in an independent conversation; if its context overflows, compression kicks in and quality degrades. Budget before dispatch:

```
estimated tokens = (brief size + required input bytes / 2 + expected output tokens + ~20k tool overhead) x 1.5 safety factor
```

Budget rules:

- **Estimate <= 100k tokens**: fine as one subtask.
- **Over 100k**: choose one -
  - split into two sequential subtasks (the first writes intermediate results to `deliverables/`, the second reads files instead of inheriting context);
  - or mandate "read on demand" in the brief: list files with absolute paths and let the subtask read selectively, never paste bulk content into the brief.
- **Keep the brief body itself under 4k tokens**: background knowledge and upstream conclusions go into files; the brief references them by absolute path. The subtask Automation prompt is even shorter - just the brief file path.

Record each task's estimate in `plan.json` under `tokenBudget.estimated`; when over budget, note the remediation in `tokenBudget.note`.

### Model tiering

Assign a `tier` to every task:

- `standard` (default): do NOT pass modelAlias when creating the Automation - it follows the system default model, same capability tier as the main conversation. Use this for anything requiring judgment, writing, or code.
- `light`: simple, mechanical, low-risk tasks (format conversion, batch renames, templated copy, data shuffling) - pick the model dynamically via `AutomationControl action:"listModels"`: prefer an alias containing `k2d8` (the current default, `k2d8-preview`, is cheaper than the standard tier); if no alias matches `k2d8`, fall back to `defaultModelAlias`. Write the actual alias chosen into `plan.json`'s `modelAlias` field to cut credit consumption.
- When in doubt, pick `standard`. Record each task's `tier` and rationale in `plan.json`.

## P2 Create subtasks

For each task:

1. Write `briefs/<task-id>.md` (template in references/brief-template.md). The brief must be self-contained: goal, absolute input paths, constraints, deliverable paths, acceptance criteria, and **progress-reporting requirements** (the subtask updates `status/<task-id>.json` at milestones).
2. Create the subtask Automation (one per task):
   - `execution.kind: "agent"`, `mode: "local_conversation"`, `workspace: { "kind": "path", "path": "<project workspace absolute path>" }` (same folder as the main conversation, so outputs stay together).
   - Keep the prompt short: read the brief file and execute. Example: `"Read <abs>/briefs/T03.md and execute it exactly. Write deliverables to the specified location and maintain status/T03.json as the brief requires."`
   - `trigger: { "kind": "manual" }` - dispatched explicitly by the orchestrator, never scheduled.
   - `result: { "kind": "conversation" }`.
   - Model: omit or set `modelAlias` per the P1 `tier`.
   - Set `timeoutMs` for long tasks.
3. Write the returned `automationId` back into the task in `plan.json`.
4. Before creating anything, give the user one combined confirmation: subtask count, parallelism, estimated credit consumption, and model tier per task.

> Slot recycling (important): each **enabled** subtask Automation occupies one cron-job slot, but a **disabled Automation keeps all run records and occupies no slot**. So total task count is not capped - only "tasks running in parallel + waiting on condition triggers at the same time" is limited by available slots:
> - Once a subtask run reaches `succeeded` (or is abandoned at a terminal state), the orchestrator **immediately disables its Automation** to free the slot, then creates/dispatches the next tasks; run records and conversationKey remain queryable.
> - Go further with "just-in-time creation": create a task's Automation only when it is about to be dispatched instead of all upfront in P2 - concurrent slot usage then roughly equals the maximum parallelism width.
> - Tasks with condition triggers must stay enabled while waiting and do occupy a slot; for long chains on a tight slot budget, use pure manual handoff (advance when the main conversation wakes up).

## P3 Dispatch & execution

**Board first, dispatch second**: P4 board setup must be complete before entering this phase (one-off, ~1-2 minutes) so the user sees live progress starting from "all pending"; only start dispatching once the board is ready.

Dispatch loop (executed by the orchestrator in the main conversation):

1. In `plan.json`, find tasks with `status: "pending"` whose `dependsOn` are all `succeeded`.
2. **Parallel**: call `AutomationControl action:"run"` for the whole independent batch at once (issue them together).
3. **Serial dependencies**: a successor fires only after its predecessor run reaches `succeeded`. Poll with `listRuns` / `readRun` - no tight loops; check once per user interaction or periodic review.
4. **Fully automatic handoff (optional, recommended for long chains)**: the main conversation only wakes on user messages, so in pure-manual mode a serial chain stalls waiting for the orchestrator to wake. For unattended progress, give the successor Automation a `condition` trigger (default: poll a Python predicate every 10 minutes): the predicate reads `plan.json` and `status/`, returning true only when all predecessors are `succeeded` and the task itself is still `pending` - the moment the predecessor finishes, the successor starts. Disable the condition trigger right after the task completes to prevent idle polling. Condition triggers are a built-in Automation capability; details live in the automation skill's `references/trigger-condition.md`.
5. **Failure handling**: run `failed`/`timeout` -> get evidence via `readRunLogs`, record `attempts` in `plan.json`, allow one rerun after fixing the brief; if it fails again, stop and report to the user - never retry indefinitely.
6. **Verification**: a subtask's self-report does not count - the orchestrator must personally check that deliverable files exist and meet the bar (read the files, not the self-report). Send back for rework if not.

After each batch, update `status` in `plan.json`; for every task reaching a terminal state (succeeded / abandoned), **immediately disable its Automation to free the slot** (records are kept - see P2 slot recycling), then create/dispatch follow-up tasks.

## P4 Kanban monitoring

**Timing: after P2 creates the subtask Automations, before P3's first dispatch** - set up the board and mount it on a canvas first, so the user has visual progress from the very first subtask. See references/status-board.md.

1. Create a kanban Widget with the Widget skill (follow the Kimi design system).
2. Create a Python status-aggregation Automation (widget task): read `plan.json` + `status/*.json` and produce an artifact with the task list / status / progress / model tier. Aggregation is pure Python with no model calls, so refresh costs almost nothing - **never** use an agent sub-conversation for periodic monitoring (each poll burns model credits, orders of magnitude more expensive than the board).
3. Bind the artifact to the Widget's `main` slot with a Binding.
4. The aggregation Automation **must use an interval trigger** (every 15-30 minutes while the project is active), and run it once manually right after setup so the board has data immediately. Never leave it manual-only - the board would never refresh itself.
5. **Place the board Widget on a standalone Dashboard canvas by default** (Canvas.placeWidget, one canvas per project) so the user can watch it outside the conversation; use Widget.show alone only when the user explicitly wants it inline in the conversation.
6. **New subtasks appear automatically**: the aggregator re-reads `plan.json` on every run, so tasks appended or re-split by the orchestrator show up on the board on the next aggregation - no changes to the board or aggregator needed.

When asked "how's it going?", the main conversation reads `plan.json` + `status/` + run states and answers in three to five lines: what is done / in progress / blocked, what is running now, and what comes next.

## P5 Aggregation & delivery

Once every task is `succeeded`:

1. Check each `deliverables` path in `plan.json` one by one - confirm every file actually exists and meets the bar (open and read it; never trust the records).
2. If needed, read subtask runs' `conversationKey` / transcripts for process conclusions.
3. Deliver a summary to the user: goal achievement, deliverables list (markdown links with absolute paths), per-subtask duration / model tier / rework, and residual risks.
4. Cleanup (ask the user first): disable or delete subtask Automations to free slots; stop the aggregation Automation's interval. Keep the orchestration files under `orchestration/` for reference.
5. **Full project deletion** (only when the user explicitly asks to delete, not archive): besides the Widget / Binding / Automation / canvas / `orchestration/<slug>/` directory, you **must also delete the subtask conversations** - deleting an Automation does NOT cascade to the local_conversation sessions it spawned, and ghost entries linger in the sidebar. How: in `daimon/agents/main/sessions/hosted-logical/conversations.sqlite`, match rows in the `conversations` table by the `localConversationTaskId` (`blueprint:<automationId>`) inside `extra_json`, delete those rows, and physically remove each row's `kernel_session_dir` directory; remind the user to switch projects or restart the client to refresh the sidebar cache.

## Hard rules

- Confirm credits first, then create any subtask Automation.
- Budget first: any task estimated over 100k tokens must be remediated before dispatch.
- Serial dependencies strictly wait for predecessor `succeeded`; never fire early on "it should be done by now".
- Subtask deliverables must be verified by the orchestrator personally; self-reports are not accepted.
- At most one automatic retry on failure; after that, ask the user.
- Slot recycling: disable a subtask's Automation immediately at terminal state (disabled keeps records, occupies no slot); total task count is uncapped - only "parallel + condition-waiting at the same time" is bounded.
- All paths absolute; briefs, status, and deliverables live under `orchestration/<project-slug>/`.

## Reference files

- `references/plan-schema.md` - full plan.json field definitions
- `references/brief-template.md` - subtask brief template
- `references/status-board.md` - status heartbeat format + kanban Widget / aggregation Automation setup
