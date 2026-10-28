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

## 相关
- [[Phases & Milestones]]
- [[Decisions]]
