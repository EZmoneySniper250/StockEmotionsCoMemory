---
tags: [usstockemotions, workflow]
updated: 2026-10-12
source: docs/dev_plan.md §1, §7
---

# Workflow

## Git
- 主干开发，`main` 必须一直是绿的；短分支命名 `feat/<表或领域>`、`fix/...`、`chore/...`
- 一个 PR = 一张表或一块完整的基建，大约 400 行以内；CI 绿 + 1 个 reviewer
- Conventional commits，例如 `feat(slv.posts): ...`
- **git 操作由 Jerry 自己做**（见 [[About Jerry]]）

## 每张表的 8 步（先写合约）
1. 合约：grain、PK、列、分区 → `contracts/<layer>/<table>.yml`（复制 `_template.yml`）或 Gold 的 dbt YAML。**写代码之前先 review**
2. Fixtures：5–20 行手写输入，覆盖边界情况 → `tests/fixtures/<table>/`
3. 先写会失败的测试 → `tests/jobs/test_<table>.py`
4. 实现：纯函数放在 `src/usse/jobs/<layer>/<table>.py`，I/O 放在很薄的 `run()` 里
5. 写入：按 PK MERGE，失败的记录进 quarantine；重跑行数不变
6. DQ 检查 → `src/usse/dq/` → `_dq_results`
7. 集成测试：`USSE_SCHEMA_PREFIX=ci_`，`pytest -m integration`
8. 文档：Bronze→Silver 写血缘说明，Gold 用 dbt docs

## 本地命令（在 `infra/local/` 下执行）
```bash
cp .env.example .env                       # 只需一次
docker compose up -d --build
docker compose exec spark python scripts/smoke_delta.py
docker compose exec spark pytest -q
docker compose exec spark ruff check src tests scripts docker
docker compose exec -w /workspace/dbt spark dbt debug
docker compose down                        # volume 保留
```
UI：JupyterLab http://localhost:8888 · RustFS 控制台 http://localhost:9001 · Spark UI http://localhost:4040

## 给 coding agent 的规则摘要（plan §11，共 20 条）
先读合约；不改 Bronze；按 PK MERGE；不在 Spark 里调 LLM；SCD2 用 as-of join；不存平均值；OBT 不加逻辑；一律 UTC；schema 名和路径从 `usse.settings` 取；所有表都是 Delta 并显式写 `USING DELTA`；DDL 从合约生成；只改自己模块的文件；notebook 只用来探索。
