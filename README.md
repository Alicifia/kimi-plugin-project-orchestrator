# Project Orchestrator

> **Copyright (c) 2026 [Alicifia](https://github.com/Alicifia). All rights reserved.**
> Released under the [MIT License](LICENSE); the copyright notice must be retained in any use or distribution. See [NOTICE](NOTICE).

A [Kimi Work](https://www.kimi.com/) plugin for **autonomous planning and multi-subtask orchestration of large projects**. It decomposes a large project into a dependency DAG of subtasks, runs each subtask in its own sub-conversation, and lets the main conversation act as the orchestrator: plan, dispatch, monitor, aggregate.

## Features

- **P0 Project analysis**: confirm goals, deliverables, and acceptance criteria; produce the project plan
- **P1 Task decomposition**: generate a task DAG (`plan.json`) with real dependencies; **context-length budgeting** - estimate each subtask's token consumption and force a split or "read on demand" when over budget, preventing subtask quality loss from context compression
- **P2 Subtask creation**: one self-contained brief per subtask, each running in an independent `local_conversation` sub-conversation sharing the main conversation's workspace folder
- **P3 Dispatch**: dependency-free tasks run in parallel; dependent tasks hand off serially after predecessors succeed (condition triggers enable unattended automatic handoff); slot recycling - finished tasks immediately free their quota slots, so total task count is uncapped
- **P4 Kanban monitoring**: a Kimi-design-system kanban Widget + Python aggregation Automation + Binding auto-refresh, showing status, progress, and model tier per subtask - the board is set up *before* dispatch and new tasks appear automatically
- **P5 Aggregation**: the orchestrator personally verifies every deliverable (never trusts subtask self-reports) and delivers a summary with file links
- **Model tiering**: complex tasks use the default model (same tier as the main conversation); simple mechanical tasks are downgraded to a lighter model to save credits

## Installation

Register/install this directory from the "Personal" tab on the Kimi Work plugins page, or install via the plugin link. Then drive it in natural language:

- "Plan and complete project XX for me - split it into parallel subtasks"
- "/project-orchestrator Expand this outline into a 20-chapter first draft"
- "How's it going?"

## Repository layout

```
project-orchestrator/
├── kimi.plugin.json                      # Plugin manifest
├── commands/
│   └── project-orchestrator.md           # /project-orchestrator slash command
├── skills/
│   └── project-orchestrator/
│       ├── SKILL.md                      # Six-phase orchestration playbook (core)
│       └── references/
│           ├── plan-schema.md            # Task DAG (plan.json) field definitions
│           ├── brief-template.md         # Subtask brief template
│           └── status-board.md           # Kanban monitoring setup guide
├── LICENSE                               # MIT License
└── NOTICE                                # Copyright notice
```

## License

Copyright (c) 2026 **[Alicifia](https://github.com/Alicifia)**. All rights reserved.

Released under the [MIT License](LICENSE). The copyright notice must be retained in any use or distribution. See [NOTICE](NOTICE).
