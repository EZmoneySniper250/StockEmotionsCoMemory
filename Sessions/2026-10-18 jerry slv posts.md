---
tags: [session, usstockemotions, silver]
author: jerry
date: 2026-10-18
updated: 2026-10-18
---

# 2026-10-18 Silver：`slv.posts` 合约和任务

## 要求
- 写 `slv.posts` 的合约，审完开始写 Silver
- Jerry 问要不要在 brz 和 slv 之间加一个 XGBoost 之类的垃圾帖模型
- 基建进度能不能可视化（公司用 Wrike，这边想要大家都能看到的）

## 决定（Jerry 2026-10-18 同意，见 [[Decisions]]）
- 合约的 5 个问题全部同意：互动数只存最新快照（不满足 PIT，Gold 回测不能用）；X 的 `includes.tweets` 不算帖子；小红书只有拉过详情的笔记才进 `posts`；作者 hash 的盐放环境变量 `USSE_AUTHOR_SALT`，三人共用同一个值；`lang` 只用来源自带的，语言检测以后再说
- **现在不上 ML 垃圾模型**：没有标注数据；现在的垃圾主要是团伙，规则更好抓；放在 brz 和 slv 之间会破坏可重建（P4）和版本化（P5）。路线：现在用规则 → M5 让 LLM 顺便输出 `is_spam` / `is_relevant`，Jaylen 抽检攒标注 → 有 1–2 千条标注后，如果规则漏得多，再训练模型，放在旁路的 `slv.post_quality_scores`（按 `model_version`）
- 进度可视化：用 [[Phases & Milestones]] 里的 Mermaid 甘特图（Obsidian 和 GitHub 都能直接渲染）。以后要任务看板再开 GitHub Projects

## 做了什么（代码仓库，未提交）
- 合约：`contracts/slv/posts.yml`（34 列）、`slv/_quarantine.yml`、`slv/_job_state.yml`
  - 审阅后改了一处：`fetch_count` 换成 `first_request_id`，因为次数在重跑时会累加，违反 P6
- `src/usse/jobs/slv/posts.py`：读 watermark 之后的 Bronze → 分来源解析成"出现记录"（一条帖子被一次调用返回一次）→ 垃圾规则 `spam_v1` → 按帖子合并 → MERGE → 最后才移动 watermark
  - X：`includes.users` 按 `author_id` 关联；cashtag = `entities` ∪ 正文正则（有些页整页没有 `entities`）；`crowd-*` 帖子按 query 里的代码算 `is_cashtag_verified`
  - 小红书：计数 `''` → null，`1.2万` → 12000
  - 没有 id、作者或发帖时间的帖子进 `_quarantine`
- `docker-compose.yml` 把 `USSE_AUTHOR_SALT` 传进 spark 容器；Jerry 本地 `.env` 里已生成一个随机值，**要放进团队密码管理器，发给 Jaylen 和 Linan**
- `docs/data_platform_plan.md` §7.1 按合约重写
- 测试：`tests/jobs/test_posts.py` 3 个（解析、去重、各种标记、隔离；重跑无变化，删掉 watermark 全量重建结果完全一致；增量只更新可变字段，`available_ts` 不后移）。全套 29 个通过，ruff 通过

## 真实数据跑出来的结果
- 665 条帖子：StockTwits 151、X 489、小红书 25；重跑写入 0；`available_ts > ingest_ts` 和 `created_ts > available_ts` 都是 0；隔离 0
- X 的 crowd：84 条 cashtag 对得上，124 条对不上（只是匹配到普通单词）→ 和 trial 2 的结论一致
- 垃圾标记：new_account 33、keyword:flai 17、cashtag_stuffing 11、duplicate_text 3（FLAI 团伙的"对不起，我还没有学会回答这个问题"）
- StockTwits 151 条里 48 条带用户自标情绪（27 看涨、21 看跌）

## 发现的问题
- **小红书笔记几乎不写 `$代码`**：25 条全部没有 cashtag。`post_mentions` 要靠中文名映射（英伟达 → NVDA），要在 M2 的 `dim_security` 里加中文别名
- **shared_author_bio 规则没抓到 FLAI 团伙**：它们的简介只是"几乎一样"（平台/平臺、标签顺序不同），精确匹配抓不到。要么做近似匹配，要么等模型
- 单轮内的规则（duplicate_text、shared_author_bio）看不到跨轮分散的团伙，全量重建时才能抓到

## 没做完 / 下一步
- `slv.post_mentions`（要先做 M2 `dim_security`，含中文别名）、`_dq_results`
- 盐值共享给 Jaylen 和 Linan
- 定时调度（24/7 主机）

## 相关
- [[2026-10-18 jerry XHS and StockTwits sim]]
- [[X Collector]]
- [[Phases & Milestones]]
