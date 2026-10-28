---
tags: [session, usstockemotions, architecture]
author: jerry
date: 2026-10-27
updated: 2026-10-27
---

# 2026-10-27 云平台选型：EMR 还是 Databricks + S3

## 要求
- 接着 2026-10-18 的进度继续（见 [[2026-10-18 jerry slv posts]]）
- Jerry 问：云架构选 EMR 还是 Databricks + S3？Databricks 的 managed connector 和 Databricks Connect 会不会调试更方便

## 讨论结论
- **managed connector（Lakeflow Connect）对我们几乎没用**：StockTwits、Reddit、X、小红书都没有现成的 connector，fetcher 照样要自己写，而且 7×24 的轮询放在 Databricks 外面的小容器里更省钱
- **真正的好处是调试和开发体验**：notebook、Spark UI、Databricks Connect（本地 IDE 打断点、远端执行）；Delta、Auto Loader、Jobs、Unity Catalog、dbt-databricks 都是原生的。EMR Serverless 是提交 job 之后去翻日志，反馈很慢
- 代价：DBU 费用、编排层有一定锁定（数据是我们自己 S3 上的 Delta，随时能带走）
- Jerry 有 DEA（Databricks Data Engineer Associate）证书

## 决定（见 [[Decisions]]）
- 现在**先写平台中立的代码**，云平台晚点定，倾向 Databricks + S3
- 可移植性规则：不用 `sparkContext` / `.rdd` / `_jvm`；只用 OSS Delta 3.2 的功能；表按名字读写；环境相关的东西只放在 `settings.py`、`spark.py`、`bootstrap.py`；合约按包内资源读

## 做了什么（代码仓库，未提交，git 由 Jerry 操作）
- 检查了现有代码：版本对应 DBR 15.4 LTS；`spark.py` 用 `getOrCreate()`；表名集中在 `settings.schema()`；没有用到 `sparkContext` 或 RDD
- `settings.py` 加了 `load_author_salt()`，`jobs/slv/posts.py` 的 `main()` 改成调用它（以后在 Databricks 上换成 secret scope 只改一处）。py_compile 通过；Docker 没开，**测试没跑**
- 将来迁移要改的地方：`settings.py` 加 `catalog`（Unity Catalog 是三段式表名）；`bootstrap.py` 换成 UC 的 managed / external location；`loader.py` 读本地 landing → 改成 Auto Loader 或用 Spark 读 S3

## 没做完 / 下一步
- trial 跑一周真实的 brz → slv，看 DBU 账单；三人的 workspace 账号；之后再改 plan 的 M8 和架构图
- 代码仓库的改动还没提交（2026-10-18 的 brz/slv 加上这次的改动）
- 下一步写代码：`slv._dq_results` → M2 `dim_security`（含中文别名）→ `slv.post_mentions`

## 后半段：`slv._dq_results`（代码仓库分支 `feat/brz-slv-posts`，未提交）
- 讨论了 DQ 和 run log 的区别：run log 看 job 跑得怎么样，DQ 看数据对不对，重点是"job 没报错，但数据错了"（比如小红书 cookie 过期，job 成功、写入 0 行）。运行计数作为 `info` 行写进 `_dq_results`，不单独建 run log 表
- Auto Loader 和 `_rescued_data` 是 Databricks 独有的，现在不用。Bronze 存原始 `payload` 字符串，已经起到 rescue 的作用；真正的漏洞是 landing 里有一行坏 JSON 会让整次 load 失败（`loader.py` 的 `json.loads`），**还没修**
- 合约 `contracts/slv/_dq_results.yml`；`contracts.py` 读取 `dq_checks`
- `src/usse/dq/`：`core.py`（`Result`、`SEVERITY`、`record`、`gate`，命令行 `python -m usse.dq.core slv.posts`，gate 关着就 exit 1）、`checks.py`（`pk_unique`、`violations`、`rate_by`、`ratio`、`volume_drop`、`run_stats`）、`slv.py`（slv 合约里每项检查的实现和阈值）
- `slv.posts` 的 `run()` 结尾跑检查，没有新数据也跑；有 error 失败时 `main()` exit 1
- 测试 46 个全过（新增 17 个），ruff 通过（`learningpath.ipynb` 有一个 import 顺序的 lint 错误，不是这次改的）
- 真实数据：先跑一次，gate 挡住了，因为 10-18 那次写入没有 DQ 记录，规则是对的。删掉 `slv.posts` 的 watermark 全量重建，665 条不变；error 检查全部 pass，垃圾帖比例 stocktwits 11%、x 10%、xhs 0%，隔离率 0；量跌检查全部 skipped（最近 4 小时没有采集）；gate open

## 第三段：Bronze DQ 和坏行隔离（`feat/brz-slv-posts`，未提交，这一波做完一起提交）
- 新合约 `contracts/brz/_quarantine.yml`：landing 里读不了的行（`malformed_json`、`bad_envelope`），主键是 `(source_table, source_file, line_no)`。不放进 `slv._quarantine`，因为那张表的 `request_id` 不能为空，而且 Bronze 不该写进 slv schema
- `loader.py`：坏行进 `brz._quarantine`，其他行照常加载；loader 每次重扫整个 landing，用主键做 anti-join，同一个坏行不会重复隔离。extractor 和合约对不上仍然直接报错（这是代码 bug，不是坏数据）。每次 load 结尾跑 Bronze DQ，写进 `slv._dq_results`（`job = loader`，`job_run_id = batch_id`）；`main()` 最后过一次 gate，关着就 exit 1
- `src/usse/dq/brz.py`：`request_id_unique`、`ok_calls_have_payload`、`fetched_le_ingest`、`result_count_matches_payload`（按表从 payload 数帖子，注意 `size(null) = -1`）、`not_null_source_post_id`、`error_rate_by_run`（本次 load 的行按采集 `run_id` 分组，阈值 0.2）、`hourly_volume_drop`（按 `result_count` 求和，所以"调用成功但返回空"也能抓到，比如小红书 cookie 刚过期的时候）
- `RunContext`、`run_checks` 挪到 `dq/core.py`，brz 和 slv 共用
- `docs/data_platform_plan.md` 升到 v1.4：§6 加 `brz._quarantine`，§7.6 写清 DQ 级别和 Gold gate
- 测试 55 个全过，ruff 通过。真实数据：4 张 Bronze 表的检查全部 pass 或 skipped，X 的 `result_count` 和 payload 对得上，landing 没有坏行

## 下一步（更新）
- Silver 只剩 `slv.post_mentions`，它要先有 M2 `dim_security`（含中文别名），Q2、Q3 没定的话先用 fixture
- 然后开始规划 Gold

## 相关
- [[Phases & Milestones]]
- [[Decisions]]
