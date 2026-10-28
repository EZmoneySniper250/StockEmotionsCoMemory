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

## 下一步（更新）
- loader 遇到坏 JSON 行，改成进 `_quarantine`，不要整次失败
- Bronze 的 DQ 检查（loader 结尾跑：`request_id_unique`、`error_rate_by_run`、`hourly_volume_drop` 等）
- M2 `dim_security`（含中文别名）→ `slv.post_mentions`

## 相关
- [[Phases & Milestones]]
- [[Decisions]]
