# 项目总控 Orchestrator（project-orchestrator）

> **Copyright (c) 2026 [Alicifia](https://github.com/Alicifia). All rights reserved.**
> 本项目以 [MIT License](LICENSE) 发布，任何使用与分发必须保留版权声明。详见 [NOTICE](NOTICE)。

一个 [Kimi Work](https://www.kimi.com/) 插件：**大型项目自主规划与多子任务编排**。把一个大型项目拆解成有依赖关系的子任务 DAG，每个子任务跑在独立的子对话里，主对话作为"总控"完成规划、派工、监控与汇总。

## 功能特性

- **P0 项目分析**：确认目标、交付物、验收标准，产出项目计划
- **P1 任务拆解**：生成任务 DAG（`plan.json`），标注真实依赖关系；**上下文长度预算**——估算每个子任务的 token 消耗，超预算强制再拆或改"按需读取"，防止子任务因上下文压缩降智
- **P2 子任务创建**：每个子任务一份自包含简报（brief），跑在独立的 `local_conversation` 子对话中，与主对话共享同一任务文件夹
- **P3 调度执行**：无依赖任务并行派发；有依赖任务等前驱成功后串行接力（支持条件触发实现无人值守全自动接力）；名额回收——完成的任务立即释放额度名额，总任务数不受上限约束
- **P4 看板监控**：Kimi 设计系统风格的看板 Widget + 状态聚合 Automation + Binding 自动刷新，实时展示每个子任务的状态、进度与模型档位
- **P5 汇总交付**：总控逐一亲自验证交付物（不接受子任务自我汇报），输出带文件链接的项目总结
- **模型分级**：复杂任务用默认模型（与主对话同级），简单机械任务自动降级到轻量模型，节省额度

## 安装

在 Kimi Work 插件页「个人」页签登记/安装本目录，或使用插件链接安装。安装后即可用自然语言驱动：

- 「帮我规划并完成一个 XX 项目，拆成子任务并行做」
- 「/project-orchestrator 把这份大纲扩写成 20 章初稿」
- 「进展如何？」

## 目录结构

```
project-orchestrator/
├── kimi.plugin.json                      # 插件清单
├── commands/
│   └── project-orchestrator.md           # /project-orchestrator 斜杠命令
├── skills/
│   └── project-orchestrator/
│       ├── SKILL.md                      # 六阶段编排 playbook（核心）
│       └── references/
│           ├── plan-schema.md            # 任务 DAG（plan.json）字段定义
│           ├── brief-template.md         # 子任务简报模板
│           └── status-board.md           # 看板监控搭建指南
├── LICENSE                               # MIT License
└── NOTICE                                # 版权声明
```

## 版权 / License

Copyright (c) 2026 **[Alicifia](https://github.com/Alicifia)**. All rights reserved.

Released under the [MIT License](LICENSE). The copyright notice must be retained in any use or distribution. See [NOTICE](NOTICE).
