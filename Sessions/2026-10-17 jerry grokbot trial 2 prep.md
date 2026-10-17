---
tags: [session, usstockemotions, grokbot, x]
author: jerry
date: 2026-10-17
updated: 2026-10-17
---

# 2026-10-17 第二轮 grokbot 试跑的准备

## 要求
- Jerry 现在有自己的 grokbot 了，要我们自己按 spec v1.3 再跑一轮

## 做了什么
- 代码仓库（未提交）：
  - `docs/grokbot_x_trial_prompt.md` 改写成 v1.3（第二轮试跑）：跑 3 轮，间隔 15 分钟，每个 query 每轮最多 2 页，总共最多 45 次调用；测 `since_id` 能不能一轮接一轮衔接上；step 0 查黑名单的 `user_id`；缺字段的页重抓一次看原因；`response_trailer`；按实际名字记录参数。白名单是 none 时跳过 kol 两路
  - 新增 `infra/local/scripts/check_grokbot_x.py`：只用标准库，自动跑 spec §8 的验收清单（字段、时间格式、`result_count`、翻页链、跨轮 `since_id`、名单版本、整页缺字段、manifest），并算出每条翻页链的覆盖时长和每分钟帖子数。用模拟数据测过：埋的 8 个错误全部抓到，合规样本 0 FAIL。ruff 还没跑（Docker 没开）
  - `.gitignore` 加了 `infra/local/data/`；试跑数据放在 `infra/local/data/landing/x_raw_search/trial2/`
- 白名单：jaylen 发来的 `samplelocalfiles/x_week_1_clean_data/x_data_week/x_follow/groups.csv`（82 个账号，`user_id` 齐全；big 47 个、small 35 个，以 10 万粉为界；来源是 jaylen 的 X 列表，2026-10-14 从 home timeline 推出来的）→ 转成 `infra/local/config/x_lists/whitelist.yml` v1，`added_on` 记 2026-10-14。试跑 prompt 已填好白名单，kol 按 big/small 分两路；big 的 kol_replies 加上黑名单会超过 1024 字符，拆成 a/b 两段；预算提到 50 次
- 待 jaylen 确认：白名单里混着新闻和资讯号（MarketWatch、zerohedge、WatcherGuru、DeItaone、unusual_whales、Stocktwits、StockMKTNewz 等），要不要加 `kind: news | kol` 把它们和真正的博主区分开
- 代码仓库已经有 `8d46060`（Jerry 提交的 spec v1.3 和 blacklist.yml）

## 没做完 / 下一步
- Jerry 把 prompt 发给 grokbot，把输出放进 `trial2/`，然后跑 `python infra/local/scripts/check_grokbot_x.py infra/local/data/landing/x_raw_search/trial2`
- 根据结果：补全 blacklist.yml 的 `user_id`；决定 `Christophep_a` 留不留；改 `x_raw_search.yml` 的列；开始写 M1 的 `x_grokbot.py` 导入
- 这个分支还没 push，PR 还没开（`gh` 还没装）

## 相关
- [[2026-10-15 jerry X blacklist and spec v1.3]]
- [[2026-10-15 jaylen grokbot X trial run]]
