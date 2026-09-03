<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License - see LICENSE and NOTICE. -->

# Kanban monitoring: status heartbeat + Widget + aggregation Automation

## 1. Status heartbeat file

`orchestration/<project-slug>/status/<task-id>.json`, updated by the subtask as its brief requires:

```json
{
  "taskId": "T03",
  "status": "running",
  "progress": 40,
  "note": "Drafting chapter 3",
  "updatedAt": "2026-09-03T21:10:00+08:00"
}
```

`status` values: `running` / `succeeded` / `failed` / `blocked`. `progress` is 0-100.
If a subtask fails to report, the orchestrator backfills the heartbeat from run records (`pending`/`running`/terminal).

## 2. Aggregation Automation (widget task, Python execution)

Create a Python Automation:

- `trigger`: `{ "kind": "interval", "every": "20m" }` while the project is active; or `manual` refreshed by the orchestrator.
- `execution.kind: "code"`, `runtime: "python"`; entry logic (write to the codeEntry path returned by create):

```python
import json, os, glob

def run(ctx):
    root = r"<abs>/orchestration/<slug>"   # substitute the real path at creation time
    with open(os.path.join(root, "plan.json"), encoding="utf-8") as f:
        plan = json.load(f)
    beats = {}
    for p in glob.glob(os.path.join(root, "status", "*.json")):
        try:
            with open(p, encoding="utf-8") as f:
                b = json.load(f)
            beats[b.get("taskId")] = b
        except Exception:
            pass
    tasks = []
    for t in plan["tasks"]:
        beat = beats.get(t["id"], {})
        tasks.append({
            "id": t["id"],
            "title": t["title"],
            "status": beat.get("status") or t["status"],
            "progress": beat.get("progress", 100 if t["status"] == "succeeded" else 0),
            "note": beat.get("note") or t.get("note", ""),
            "tier": t.get("tier", "standard"),
            "dependsOn": t.get("dependsOn", []),
        })
    done = sum(1 for x in tasks if x["status"] == "succeeded")
    return {"artifact": {
        "projectTitle": plan["project"]["title"],
        "total": len(tasks), "done": done,
        "tasks": tasks,
    }}
```

- `result.kind: "artifact"`, with a schema matching the structure above (`properties` include `projectTitle/total/done/tasks`; `tasks.items` carry the fields above; `additionalProperties: true`).

## 3. Kanban Widget

Create it with the Widget skill (read widgetdesign first and follow the Kimi design system); `slots.main` matches the artifact shape above. Suggested layout:

- Header: project title + overall progress (done/total progress bar).
- Body: kanban columns by status (pending / running / done / failed-blocked); each card shows task id, title, progress bar, model-tier badge (standard/light), and the latest note.
- Use a visible but restrained warning color for failed/blocked.

## 4. Binding

Use the Binding skill to bind the aggregation Automation's artifact to the Widget's `main` slot. Every successful aggregation run then refreshes the board automatically; the orchestrator can also `run` it manually at key moments for an instant refresh.

## 5. After the project ends

In P5, with user confirmation: stop the interval (disable the aggregation Automation); keep the board as an archive or delete it together with the Widget.
