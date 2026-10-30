---
tags: [usstockemotions, data-model]
updated: 2026-10-29
source: docs/data_platform_plan.md §2 §4 §6–§8
---

# Data Model

> 摘要。完整的列定义在 `docs/data_platform_plan.md` §6–§8；Bronze/Silver 的 schema 以 `infra/local/contracts/<layer>/<table>.yml` 为准，Gold 以 dbt 模型 YAML（`contract: enforced`）为准。

## 设计原则（必须遵守）
| # | 原则 |
|---|---|
| P1 | 先声明 grain（"一行 = ..."）再写代码 |
| P2 | 三个时间戳：`created_ts`、`ingest_ts`、`available_ts`，一律 UTC |
| P3 | Point-in-time：时刻 T 的特征只能用 `ingest_ts/available_ts <= T` 的数据 |
| P4 | Bronze 不可变（只追加），Silver/Gold 能从 Bronze 完整重建 |
| P5 | 标签、score、规则都带版本，新版本追加，不覆盖 |
| P6 | 幂等写入：Silver/Gold 一律按 PK MERGE |
| P7 | LLM 只做内容分类，不预测市场影响 |
| P8 | 规则做决定，agent 只解释 |
| P9 | Agent 只读 Gold |
| P10 | 事实表存可加的量（sum/count），不存平均值和比率 |
| P11 | 不在 Spark 里逐行调 LLM，打标签是单独的批任务 |
| P12 | 失败的记录放进 `_quarantine`，不丢弃 |

## Bronze（`brz`）：只追加，按 `ingest_date` 分区，允许重复，**一次 API 响应一行**（2026-10-12 起）
命名 `brz.<source>_<purpose>`。共有列：`payload`（整个响应原样）、`api_endpoint`、`request_params`（可空）、`ingest_ts`、`batch_id`、`ingest_date`。帖子的展开、和作者的关联都放 Silver。

| 表 | Grain | 额外列 | 合约 |
|---|---|---|---|
| `stocktwits_raw` | 一次 stream 响应（≤30 条） | `request_symbol`、`result_count`、`newest_id`、`oldest_id`、`has_more` | ✅ |
| `xhs_raw_search` | 一页搜索结果（约 20 条） | `search_keyword`、`sort_type`、`page_no`、`result_count`、`has_more` | ✅ |
| `xhs_raw_note` | 一次笔记详情响应 | `source_post_id` | ✅ |
| `x_raw_search` | 一页 X 搜索（≤25 条） | `collector`、`request_query`、`result_count`、`newest_id`、`oldest_id`、`next_token` | ✅ |
| `reddit_raw` | 一次 listing 响应 | `subreddit` | 未写 |
| `prices_raw` | 一次供应商响应 | 等 Q6 | 未写 |
| `llm_label_raw` | 打标签任务的一次 LLM 调用（多条帖子） | `label_version`、`post_ids`、token 数 | 未写（M5，2026-10-29） |
| `_ingest_state` | 一个 source × endpoint 的 cursor | | 未写 |

## Silver（`slv`）：以实体为中心，接近 3NF，按 PK MERGE
| 表 | Grain | 要点 |
|---|---|---|
| `posts` | 一个唯一的帖子 | PK = source + source_post_id；`author_hash`（不存原始 ID）；`ingest_ts` 取最早，会变的字段取最新 |
| `post_mentions` | 帖子 × 实体（桥接表） | cashtag / ETF map / LLM；ticker→FIGI 用 **as-of join** 关联 `dim_security`；映射不上的进 quarantine |
| `post_labels` | 帖子 × 实体 × label_version | sentiment −1..1、stance、topic（等 Q5）；由 LLM 打标签任务生成，从 `brz.llm_label_raw` 解析，能从 Bronze 重建；StockTwits 用户标签只用来验证（2026-10-29） |
| `dim_security` | 一只证券 × 一个有效期 | SCD2，FIGI |
| `dim_sector`、`etf_map` | | Q3 未定 |
| `prices_1h` | symbol × 小时 | |
| `_quarantine`、`_dq_results` | | 运维用的表 |

## Gold（`gld`）：Kimball，dbt-spark
- 原子事实表 `fct_post_sentiment`（帖子 × 实体 × 版本）→ 三张汇总表 `agg_{ticker,sector,market}_sentiment_4h`（实体 × 4h bucket × source × 版本）
- 行业和市场汇总时 **count distinct `post_id`**（同一个帖子提到同一行业的多只股票，只算一次）
- `fct_price_4h`、`fct_score`（存各个分项，带 `available_ts`）、`fct_alert`、`fct_alert_outcome`（单独成表，避免泄漏到触发逻辑）、`fct_agent_run`
- 维度表：`dim_date`、`dim_time_bucket`（Q4）、`dim_entity`、`dim_security`、`dim_sector`、`dim_source`、`dim_label_version`、`dim_topic`、`dim_score_model`、`dim_alert_rule`
- `obt_ticker_4h`：只从 Gold 构建，只做 join，不加业务逻辑；`_train` 变体不能暴露给 agent

## 命名
schema 用 `brz/slv/gld`，表名前缀 `raw_`/`dim_`/`fct_`/`agg_`/`obt_`/`vw_`/`_`，列名后缀 `*_ts`(UTC)/`*_id`/`*_key`/`*_hash`/`*_count`/`*_sum`/`is_*`。

相关：[[Architecture Diagrams]] · [[Decisions]]
