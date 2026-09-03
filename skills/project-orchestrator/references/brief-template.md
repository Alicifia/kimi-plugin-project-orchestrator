<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License — see LICENSE and NOTICE. -->

# 子任务简报模板

路径：`orchestration/<项目slug>/briefs/<task-id>.md`

简报是子任务唯一的任务书，必须**自包含**——子任务看不到主对话历史。正文控制在 4k tokens 以内：长背景一律落成文件，简报里只给绝对路径。

```markdown
# 子任务 T03：<标题>

## 你要做什么
<一句话目标 + 2-5 条具体要求>

## 背景
<最少必要背景。更多背景见：<abs>/orchestration/<slug>/context/xxx.md>

## 输入
- <输入文件1 绝对路径> — 你需要从中取什么
- <输入目录 绝对路径> — 按需选读，不要全量读入

## 约束
- <风格/格式/技术栈/禁止事项>
- 上下文自律：输入文件按需选读；如果必须读的内容超过约 8 万 tokens，先读目录/清单做取舍，不要一口气全读。

## 交付物
把结果写入：<abs>/orchestration/<slug>/deliverables/<T03-产物文件名>
<格式要求>

## 验收标准
- <总控会逐条核对的客观标准>

## 进度上报（必做）
开始工作时、每完成一个里程碑、以及结束时，更新 <abs>/orchestration/<slug>/status/T03.json：
{ "taskId": "T03", "status": "running", "progress": 40, "note": "当前在做什么", "updatedAt": "<ISO时间>" }
结束时 status 写 "succeeded" 或 "failed"，progress 写 100 或实际进度。
```

## 给子任务 Automation 的 prompt（保持短）

```
阅读 <abs>/orchestration/<slug>/briefs/T03.md，严格按简报执行。
交付物写到简报指定位置，并按简报要求维护 status/T03.json。
完成后用 3-5 句话总结：做了什么、交付物路径、遇到的问题。
```
