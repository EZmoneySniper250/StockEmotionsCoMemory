---
tags: [usstockemotions, roadmap]
updated: 2026-10-18
source: docs/dev_plan.md §2, docs/data_platform_plan.md §1.3 §10
---

# Phases & Milestones

## 产品阶段
| Phase | 范围 | 状态 |
|---|---|---|
| **Phase 1（当前）** | StockTwits + Reddit 帖子 → LLM 标签 → 市场/行业/个股的 4h 指标 → score → 告警 + agent 解释；小时级价格用来评估 | 进行中 |
| Phase 2 | 历史 tick 回放（引入 Kafka）、恐慌指数（VIX、期限结构）、EPU、Trump 发帖事件研究 | 计划中 |
| Phase 3 | 模拟盘（Alpaca）、PnL 评估 | 计划中 |

## 基建里程碑（Jerry）
排期是指示性的，从 2026-10-12 开始按周算。和 plan §10 原顺序比，有一处改动：**Bronze 采集提到最前面**，因为 StockTwits 拿不到历史数据。

| M | 内容 | Plan step | 状态 | 被谁卡住 |
|---|---|---|---|---|
| M0 | 本地环境（Spark + RustFS + metastore + Delta + dbt + CI） | 1 | ✅ 2026-10-10 完成 | — |
| **M1** | Bronze 采集，7×24 运行：`tables.py`（根据合约生成 DDL）、envelope + `RunWriter`（即 sink）、StockTwits/小红书 fetcher、通用 `loader.py`；grokbot 直接写 envelope。落地目录是 `<lake>/landing/<table>/`，**不能放在 `<lake>/brz/` 的表目录里**（VACUUM 会删掉 Delta log 没记录的文件） | 2 | 🟡 代码完成（jerry 2026-10-18，26 个测试，本地端到端跑通）；还差：24/7 主机上的定时调度、Reddit fetcher | Q7（Reddit）、Q8（主机） |
| M2 | 参考数据：`dim_security`（SCD2，FIGI）、`dim_sector`、`etf_map` | 3 | 未开始 | Q2、Q3（SCD2 机制可以先用 fixture 做） |
| M3 | Silver：`posts`、`post_mentions`（规则部分）、`_quarantine`、`_dq_results`、按 watermark 增量处理 | 4、5 | 🟡 进行中（jerry 2026-10-18：`slv.posts` 完成，真实数据 665 条；jerry 2026-10-27：`_dq_results` + Gold gate + Bronze DQ + `brz._quarantine` 完成；还差 `post_mentions`（等 M2）） | `post_mentions` 依赖 M2 |
| M4 | Gold 薄切片（dbt）：维度表、`fct_post_sentiment`、三张 `agg_*_4h` | 6 | 未开始 | Q4、Q9 |
| M5 | LLM 打标签任务 → `post_labels` | 7 | 未开始 | Q5 |
| M6 | 价格、`fct_score`、`fct_alert`、`fct_alert_outcome` | 8、9 | 未开始 | Q1、Q6 |
| M7 | `obt_ticker_4h`、agent 视图、`fct_agent_run` | 10 | 未开始 | — |
| M8 | Terraform 部署到 AWS（EMR Serverless、Glue、S3、IAM） | 11 | 未开始 | — |

```mermaid
gantt
    title Data infra progress (updated 2026-10-18)
    dateFormat YYYY-MM-DD
    axisFormat %m-%d
    todayMarker on
    section Foundation
    M0 Local stack                        :done, m0, 2026-10-10, 1d
    section M1 Bronze
    Contracts + envelope v1               :done, m1a, 2026-10-12, 2026-10-18
    Collectors + loader (ST, XHS)         :done, m1b, 2026-10-18, 1d
    24/7 host + scheduler (Q8)            :active, m1c, 2026-10-19, 5d
    Reddit fetcher (Q7)                   :m1d, after m1c, 3d
    section X via grokbot
    Trial 1 (jaylen)                      :done, x1, 2026-10-15, 1d
    Trial 2 + spec v1.4 freeze            :done, x2, 2026-10-18, 1d
    Production prompt (jaylen)            :active, x3, 2026-10-19, 5d
    section Reference
    M2 dim_security, sectors (Q2, Q3)     :m2, 2026-10-21, 7d
    section Thin slice
    M3 slv.posts                          :done, m3a, 2026-10-18, 1d
    M3 DQ + gate                          :done, m3c, 2026-10-27, 1d
    M3 post_mentions                      :m3b, after m2, 5d
    M4 Gold thin slice (Q4, Q9)           :m4, after m3b, 10d
    section Depth
    M5 LLM labeling (Q5)                  :m5, after m3a, 10d
    M6 Prices, score, alerts (Q1, Q6)     :m6, after m4, 14d
    M7 OBT, agent views                   :m7, after m6, 7d
    section Cloud
    M8 Terraform AWS deploy               :m8, after m7, 14d
```

图例：`done` 灰色 = 完成，`active` 高亮 = 进行中，其余 = 计划中；红线是今天。日期是指示性的，更新状态时改这张图和上面的表（谁做完谁改，标上作者）。

## 近期计划：brz + slv（2026-10-12 讨论过；brz 部分已按 envelope 方案完成，见 [[2026-10-18 jerry XHS and StockTwits sim]]）
**PR #1 `feat/brz-ingestion`**
1. `src/usse/tables.py`：`ensure_table(contract_path)`，读合约生成 `CREATE TABLE IF NOT EXISTS ... USING DELTA`。brz 和 slv 都要用，先做
2. 合约：`brz/_ingest_state.yml`、`brz/raw_reddit.yml`
3. `ingestion/sink.py`：`Sink` 协议 + `S3JsonSink`，写到 `ingest_date=.../<batch_id>.jsonl`
4. `ingestion/state.py`：读写 `_ingest_state`，重启后从 cursor 继续
5. `ingestion/stocktwits.py`：只负责抓取，记录外面包上元数据；HTTP 用 mock 测试；做限流和退避
6. Spark append 任务：JSONL → `brz.raw_stocktwits`

**PR #2 `feat/slv-posts`**
7. 合约：`slv/posts.yml`、`slv/_quarantine.yml`、`slv/_dq_results.yml`
8. `jobs/slv/posts.py`，写成纯函数：解析 → 转 UTC → `author_hash`（加盐 SHA-256）→ 去重（`ingest_ts` 取最早，会变的字段取最新）→ 垃圾帖标记 → 按 `post_id` MERGE
9. DQ：PK 唯一、`created_ts <= ingest_ts`；同一天重跑行数不变

先不做 `post_mentions` 的 as-of join，要等 M2。

**开工前要先验证的风险**：StockTwits 不再接受新开发者注册，无认证的 `streams/symbol/*.json` 可能被 Cloudflare 拦；Reddit 新应用可能要先审批。都**还没验证**，先用 curl 打一下。
