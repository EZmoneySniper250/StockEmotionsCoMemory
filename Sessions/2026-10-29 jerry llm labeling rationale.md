---
tags: [session, usstockemotions]
author: jerry
date: 2026-10-29
updated: 2026-10-29
---

# 2026-10-29 为什么要用 LLM 打标签

## 要求
- 讨论 Gold 怎么设计，以及 Gold 之前还差哪几步。
- Jerry 的要求：不能白上 LLM，先要有理由和数据；把数字和理由写进 vault；架构先写出来，不写具体用哪个模型；把这一步提到 Gold 前面，所有 plan 一起改。

## 做了什么
- 盘点 Gold 前还差什么：`slv.post_labels` 还不存在，`fct_post_sentiment` 的粒度是帖子 × 实体 × label_version，没有标签就做不出来；另外还有 Q4（4h 桶）、Q3（行业）、dbt sources 是空的、CI 里还没有 `dbt build`。
- 在 2026-10-18 的模拟采集数据上统计（`slv.posts` + `slv.post_mentions`，只算映射到 ticker / market 的、非垃圾帖）：

| 指标 | 结果 |
|---|---|
| StockTwits 带用户标签的帖子 | 47 / 134 = **35%**（看多 27，看空 20）；疑似垃圾帖 1 / 17 |
| 各来源占比 | StockTwits 60%（134）、X 33%（73）、小红书 8%（17） |
| 只靠用户标签，有情绪的帖子 | 47 / 224 = **21%** |
| 每条帖子平均对应几个实体 | StockTwits 2.5、X 1.7 |
| 第 3 个数：LLM 和用户标签的一致率 | **还没测**，等 M5 试跑 |

- 样本很小：只有一次采集，X 只覆盖 03:29–03:59 UTC 半小时，小红书 25 条。X 采到 489 条，只有 73 条映射上（crowd 流按 cashtag 过滤掉的，加上不在临时股票池里的票）。
- 代码仓库的 docs 改了，**还没提交**（代码仓库的 git 由 Jerry 自己操作）：
  - `docs/data_platform_plan.md` 升到 v1.5：§7.3 重写（为什么用 LLM、实测数字、架构、试跑），§6 加 `brz.llm_label_raw`，§3 的 LLM 改成不指定供应商，§10 顺序，§12 加两条决定，Q9 关闭，加 Q16
  - `docs/dev_plan.md` 升到 v1.1：M5 排在 M4 前面（编号不变），M5 的交付物和验收标准重写，甘特图、问题表、编排、风险跟着改
  - `docs/architecture.md`：数据流图和批处理时序图加上 `llm_label_raw`，LLM 供应商改成待定
  - `infra/local/contracts/slv/posts.yml`：`user_sentiment_tag` 的说明改成"验证 LLM 用"
  - pytest 66 个全部通过（只改了文档，确认没有误动代码）

## 理由（一句话）
情绪不够：只靠用户标签，约 8 成的帖子没有情绪，X 和小红书一条都没有，而且帖子级的标签分不开同一条帖子里对不同股票的看法。所以要用 LLM 给所有来源打标签，StockTwits 的用户标签改成检验 LLM 准不准的标准答案。

## 架构要点（不绑定模型）
- 选任务 → 多条帖子拼一次调用 → `LabelClient` 接口 → envelope 写进 landing → loader → `brz.llm_label_raw` → Silver 解析、校验 → MERGE 进 `slv.post_labels`，不合格的进 `_quarantine`。
- 原始响应存在 Bronze：LLM 输出不可复现而且花钱，存下来 `post_labels` 就能从 Bronze 重建（P4），解析出错也不用花钱重新打。
- `label_version` = 名字 + (prompt, 输出 schema, 模型, 参数) 的 hash。`topic` 等 Q5 定了再加，不挡 M5。
- PIT：标签的 `available_ts` 用帖子的 `available_ts`，`labeled_ts` 也留着，回测想更严格可以按它过滤。
- 上线前先试跑：约 200 条带用户标签的 StockTwits 帖子，门槛事先定好；上线后一致率作为每次运行的 `warn` 检查。
- 完整版在代码仓库 `docs/data_platform_plan.md` §7.3。

## 决定
- 情绪标签由 LLM 给所有来源打，M5 提到 M4 前面；不用 `user_tag_v0`（Q9 关闭）。
- LLM 打标签的架构：原始响应进 `brz.llm_label_raw`，模型和供应商待定（Q16）。
- （都已同步到 [[Decisions]]）

## 没做完 / 下一步
- Jerry 审代码仓库里 docs 的改动，然后自己提交。
- Q16：选模型、定试跑一致率的门槛。
- M5 按 8 步流程开工：先写 `brz/llm_label_raw.yml` 和 `slv/post_labels.yml` 两份合约；prompt 和输出 schema 放 `labeling/`，要 Linan 和 Jaylen 审。
- 采集真正 7×24 跑起来以后，重新统计这几个比例。
- Gold 薄切片（M4）等 M5 之后再做，还要先定 Q4。

## 相关
- [[Decisions]] · [[Open Questions]] · [[Phases & Milestones]] · [[Data Model]]
