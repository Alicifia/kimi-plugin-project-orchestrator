<!-- Copyright (c) 2026 Alicifia (https://github.com/Alicifia). All rights reserved. Licensed under the MIT License - see LICENSE and NOTICE. -->

# Subtask brief template

Path: `orchestration/<project-slug>/briefs/<task-id>.md`

The brief is the subtask's only assignment document and must be **self-contained** - the subtask cannot see the main conversation's history. Keep the body under 4k tokens: long background goes into files; the brief references them by absolute path.

```markdown
# Subtask T03: <title>

## What you must do
<One-sentence goal + 2-5 concrete requirements>

## Background
<Minimum necessary background. More context: <abs>/orchestration/<slug>/context/xxx.md>

## Inputs
- <absolute path of input file 1> - what you need from it
- <absolute path of input directory> - read selectively, do not load everything

## Constraints
- <style / format / tech stack / prohibitions>
- Context discipline: read inputs selectively; if required reading exceeds ~80k tokens, scan the directory/listing first and pick - never bulk-read everything at once.

## Deliverable
Write the result to: <abs>/orchestration/<slug>/deliverables/<T03-output-filename>
<format requirements>

## Acceptance criteria
- <Objective criteria the orchestrator will check one by one>

## Progress reporting (mandatory)
When starting, at each milestone, and when finished, update <abs>/orchestration/<slug>/status/T03.json:
{ "taskId": "T03", "status": "running", "progress": 40, "note": "what you are doing now", "updatedAt": "<ISO timestamp>" }
On finish, set status to "succeeded" or "failed", progress to 100 or the actual value.
```

## Prompt for the subtask Automation (keep it short)

```
Read <abs>/orchestration/<slug>/briefs/T03.md and execute it exactly.
Write deliverables to the location specified in the brief and maintain status/T03.json as required.
When done, summarize in 3-5 sentences: what was done, deliverable paths, any problems encountered.
```
