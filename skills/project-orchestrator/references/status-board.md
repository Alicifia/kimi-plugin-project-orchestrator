<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License — see LICENSE and NOTICE. -->

# 看板监控：状态心跳 + Widget + 聚合 Automation

## 1. 状态心跳文件

`orchestration/<项目slug>/status/<task-id>.json`，由子任务按简报要求更新：

```json
{
  "taskId": "T03",
  "status": "running",
  "progress": 40,
  "note": "正在生成第三章",
  "updatedAt": "2026-09-03T21:10:00+08:00"
}
```

`status` 取值：`running` / `succeeded` / `failed` / `blocked`。`progress` 0–100。
子任务不更新时，总控以 run 记录为准（`pending`/`running`/终态）补写心跳。

## 2. 聚合 Automation（widget task，Python 执行）

创建一个 Python Automation：

- `trigger`：项目活跃期 `{ "kind": "interval", "every": "20m" }`；或 `manual` 由总控刷新。
- `execution.kind: "code"`, `runtime: "python"`，入口逻辑（写入 create 返回的 codeEntry 路径）：

```python
import json, os, glob

def run(ctx):
    root = r"<abs>/orchestration/<slug>"   # 创建时替换成真实路径
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

- `result.kind: "artifact"`，schema 与上面返回结构对应（`properties` 含 `projectTitle/total/done/tasks`，`tasks.items` 含上述字段，`additionalProperties: true`）。

## 3. 看板 Widget

用 Widget 技能创建（先读 widgetdesign 遵循 Kimi 设计系统），`slots.main` 的数据形态与上面 artifact 一致。界面建议：

- 顶部：项目标题 + 总进度（done/total 进度条）。
- 主体：按状态分栏的看板（待办 / 进行中 / 已完成 / 失败阻塞），每张卡片显示任务 id、标题、进度条、模型档位角标（standard/light）、最新 note。
- 失败/阻塞用醒目但克制的警示色。

## 4. Binding

用 Binding 技能把聚合 Automation 的 artifact 绑到 Widget 的 `main` slot。之后聚合每次成功运行，看板自动刷新；总控在关键节点也可手动 `run` 一次立即刷新。

## 5. 项目结束后

P5 阶段经用户确认后：停掉 interval（disable 聚合 Automation），看板留作存档或随 Widget 一并删除。
