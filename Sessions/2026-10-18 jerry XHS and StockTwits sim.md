---
tags: [session, usstockemotions, stocktwits, xhs, bronze]
author: jerry
date: 2026-10-18
updated: 2026-10-18
---

# 2026-10-18 小红书和 StockTwits 模拟采集、Bronze 合约改成 envelope、修 CI

## 要求
- 模拟抓一轮小红书和 StockTwits，然后完善 Bronze
- Jerry 同意四份 Bronze 合约统一按 envelope 改
- 上一次 merge 后 CI 失败了，查原因

## 模拟采集（2026-10-18 04:35 UTC）
- 脚本在 Claude 的临时目录（`sim_collect.py`），还不是正式代码；输出在代码仓库 `infra/local/data/landing/{stocktwits_raw,xhs_raw_search,xhs_raw_note}/sim1/`，格式就是 envelope v1，每轮带 manifest
- **StockTwits**：SPY、QQQ、NVDA、TSLA、AAPL 各 1 页，2 分钟后用 `since` 再抓一轮；10 次调用 8 次成功，133 条唯一消息
  - Cloudflare 偶尔拦（403 "Just a moment..."），带了浏览器 UA 也会，每轮 5 次里约 1 次 → 要退避重试
  - **`since` 增量不漏**：用一个落后很多的 `since` 测过，返回的是紧跟在它后面的 30 条，`cursor.more=true` 表示还有；沿 `cursor.since` 翻到 `more=false` 就是全量
  - 34% 的消息带用户自标 Bullish/Bearish，可以给 `user_tag_v0`（Q9）用
  - 一条消息常带 2–5 个代码，会出现在每个代码的 stream 里
- **小红书**：不用起后端，直接用 XHS_ALL_IN_ONE 的 Python 客户端（`XHS_Apis` + `XHSPcAuth.from_cookie`，cookie 来自 `D:\xhs_all_in_one\xhs_cookie.txt.txt`，2026-10-12 导入的仍有效）。5 个关键词各搜 1 页，每个取前 5 条拉详情，30 次调用全部成功，没被风控
  - "按最新排序"不严格：一条 10-11 的笔记 10-17 更新过，也被排上来
  - 同一条笔记在不同关键词下重复出现 → 同一轮内先按笔记 id 去重，再拉详情
  - 搜索结果里混的是 `rec_query`（之前记的是 `hot_query`）
  - 详情返回里有服务器时间 `data.current_time`

## Bronze 合约（代码仓库，未提交）
- 四份合约（`stocktwits_raw`、`xhs_raw_search`、`xhs_raw_note`、`x_raw_search`）统一改成：**一行 = 一次调用 = 一个 envelope**
  - 共有列一一对应 envelope 字段：`payload`、`collector`、`collector_version`、`run_id`、`request_id`、`query_id`、`page_index`、`api_endpoint`、`request_params`、`call_status`、`error_message`、`http_status`、`response_format`、`requested_ts`、`fetched_ts`、`notes`
  - 加载元数据：`source_file`、`ingest_ts`、`batch_id`、`ingest_date`
  - **PIT 用 `fetched_ts`**，`ingest_ts` 是 loader 写入时间
  - **失败调用也存**，`payload` 可为空
  - `xhs_raw_search` 的 `search_keyword` 改名为 `request_query`
  - `x_raw_search` 加了名单版本、`response_trailer`、`quota`
  - dq 新增 `request_id_unique`、`ok_calls_have_payload`、`fetched_le_ingest`、`error_rate_by_run`
- `docs/data_platform_plan.md` 升 v1.3（§6 改成 envelope、失败调用也存、新共有列）
- `docs/dev_plan.md` M1：sink 写 envelope；fetcher 每次调用都出 envelope；xhs 直接用 Python 客户端；grokbot 不用导入程序只做校验；新增一个通用 loader `loader.py`

## CI 失败的原因
- 2026-10-18 的两次 run（PR 和 merge 到 main）都死在 **Lint**：`scripts/check_grokbot_x.py` 有 2 行超过 110 字符（E501）。这个脚本是我写的，当时 ruff 没装、没跑 lint
- `requirements.txt` 里是 `ruff>=0.8`，没锁版本，CI 装到的是 0.17.1；本地用同版本跑 `ruff check src tests scripts docker` 已全部通过
- 顺手修了检查脚本：名单版本只要在 v1 到当前版本之间就算对（黑名单升到 v2 后，trial 2 的 v1 数据曾误报 30 个 FAIL）
- 单元测试本地没跑（本机没装 pyspark）；CI 上这次因为 lint 失败被跳过了，下次 push 才会跑到

## 发现的问题
- **trial 2 的 X 原始数据已经提交进了 GitHub**（`infra/local/data/landing/x_raw_search/trial2/`，在 `c2ee29e` 里），起因是 2026-10-17 按 Jerry 的要求从 `.gitignore` 去掉了 `infra/local/data/`。代码仓库是公开的（不用登录就能调 Actions API），等于把别人的帖子公开转发了。已经告诉 Jerry，等他决定怎么处理

## 决定
- 见 [[Decisions]] 2026-10-18：所有 Bronze 表的输入统一为 envelope v1，失败调用也存，PIT 用 `fetched_ts`

## 没做完 / 下一步
- 写正式代码：`sink.py`、`tables.py`、fetcher（stocktwits、xhs）、`loader.py`
- 锁定 ruff 版本，避免 CI 随新版本突然变红
- 原始数据进 git 的问题等 Jerry 决定

## 相关
- [[X Collector]]
- [[2026-10-12 XHS connectivity test]]
- [[2026-10-18 jerry grokbot trial 2 results]]
