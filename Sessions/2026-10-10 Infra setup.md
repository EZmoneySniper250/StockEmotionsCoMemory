---
tags: [session]
project: USStockEmotions
date: 2026-10-10
---

# 2026-10-10 Infra setup（M0）

## 用户要求
1. 读 data platform plan，熟悉项目，模拟搭建本地环境，画基建图，写详细的开发计划
2. 所有东西归到 `infra/local/`；加 `.gitignore`（**repo 由 Jerry 自己 init，Claude 不要动**）；根据实际搭出来的东西更新 `docs/` 里的 plan；解释以后 schema 写在哪里
3. 确认表是不是 Delta → 加上"默认 Delta"的设置和规则；按人分模块建文件夹
4. 概念问答：Delta 和 T-SQL/Postgres 的区别、并发控制、Spark 代码写在哪、能不能用 ipynb、日常工作流程、怎么迁移到 EMR → 整理进 `docs/deltaprinciples.md`
5. 把 Jupyter 装进 Docker（前提是测试确实离不开 Docker；结论是离不开）

## 做了什么
- docker compose：RustFS（storage）、Postgres 16 Hive metastore、spark 容器（PySpark 3.5.3 + Delta 3.2.1 + dbt-spark + JupyterLab 4.5）
- 冒烟测试 `scripts/smoke_delta.py`（schema、MERGE 幂等、time travel、跨 session 读 catalog）连续两次通过；pytest 4/4；ruff 无报错；`dbt debug` 正常
- `docs/architecture.md`（4 张 Mermaid 图 + `img/data_flow.png`）、`docs/dev_plan.md`（M0–M8）、plan 升到 v1.1（RustFS、§3.4 仓库结构、新增 Rule 14–20、Q8/Q9）
- 合约模板 + `contracts/brz/raw_stocktwits.yml`
- 模块文件夹 `labeling/`、`modeling/`、`research/`，各带 README
- `notebooks/00_getting_started.ipynb`，在容器里实际跑过
- Jerry 之后自己 init 了 repo，提交了两次：`9107a30`、`f4c1b41`

## 决定
见 [[Decisions]] 里 2026-10-10 的各条（RustFS、预导入 metastore 表结构、默认 Delta、模块化、容器内 Jupyter 等）

## 排查过程中遇到的坑
- 冒烟测试卡住 → JVM thread dump 显示 metastore 死锁 → 改成预导入表结构
- SeaweedFS 删表后留空目录 → 换成 RustFS
- 都已记入 [[Gotchas]]

## 下一步
- M1 Bronze 采集；团队要先定 Q2、Q7、Q8
