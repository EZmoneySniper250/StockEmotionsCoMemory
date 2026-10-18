---
tags: [project, moc, usstockemotions]
updated: 2026-10-18
repo: D:\USStockEmotions
---

# USStockEmotions — 入口

> 任何 AI session 在这个项目上动手前，先读 [[AGENTS]]（读写规则和 git 规则），再读这一页，然后按需读下面的笔记。人类的设置说明见 [[README]]。

## 一句话
把散户在社交媒体上的讨论（StockTwits、Reddit、X、小红书）变成**可回测、point-in-time 正确的情绪信号**，覆盖市场、行业、个股三个粒度；规则触发告警，agent 只负责解释告警（**rules decide, the agent explains**）。

## 当前状态（2026-10-12）
- **M0 本地环境：完成**（2026-10-10）
- **下一步：M1 Bronze 采集**（关键路径），然后是 M3 Silver `posts`
- 业务代码还没开始写。2026-10-12 写好了 4 份 Bronze 合约（`stocktwits_raw`、`xhs_raw_search`、`xhs_raw_note`、`x_raw_search`），plan 升到 v1.2（未提交，git 由 Jerry 操作）
- 详情见 [[Phases & Milestones]]

## 笔记
| 笔记 | 内容 |
|---|---|
| [[Phases & Milestones]] | Phase 1/2/3，M0–M8 的状态、产出、阻塞项 |
| [[Architecture Diagrams]] | 数据流、本地运行时、环境隔离、批处理时序（Mermaid） |
| [[Data Model]] | brz / slv / gld 各层的表、粒度、设计原则 P1–P12 |
| [[Decisions]] | 决策日志，每条都写了原因 |
| [[Open Questions]] | Q1–Q9 及谁负责 |
| [[Team & Ownership]] | 谁负责哪个文件夹 |
| [[Gotchas]] | 环境的坑和怪癖 |
| [[X Collector]] | grokbot X 采集：冻结的接口、可以改的部分、当前版本、交给 jaylen 的 prompt 任务 |
| [[Workflow]] | 每张表的 8 步流程、git 流程、本地命令 |
| `Sessions/` | 每个 session 的记录 |

## 仓库里的规格（唯一来源，不要在 vault 里分叉）
- `docs/data_platform_plan.md`：做**什么**（v1.1）
- `docs/dev_plan.md`：**怎么做、按什么顺序**
- `docs/architecture.md`：图
- `docs/deltaprinciples.md`：Delta 概念问答（用 T-SQL 类比）
- `infra/local/README.md`：怎么跑本地环境
- `docs/grokbot_x_spec.md`：给 grokbot 的 X 采集输出规范 v1.4（2026-10-18），§0 是冻结的接口，改 prompt 前必读（摘要见 [[X Collector]]）
- `docs/grokbot_x_trial_prompt.md`：grokbot 试跑 prompt v1.4（2026-10-18，未实测，交给 jaylen）

## Sessions
`Sessions/` 文件夹里每个 session 一篇，文件名是 `YYYY-MM-DD <作者> <主题>`，按文件名排序就是时间顺序。这里**不再手动维护列表**，因为多人同时追加容易冲突。
