---
tags: [session, usstockemotions, grokbot, x]
author: jerry
date: 2026-10-15
updated: 2026-10-15
---

# 2026-10-15 X 黑名单和 grokbot spec v1.3

## 要求
- 评审 jaylen 试跑提的黑名单建议，看试跑结论对我们的数据库是否适用（[[2026-10-15 jaylen grokbot X trial run]]）
- 黑名单和 spec v1.3 按评审结论改，先写好，不提 PR

## 做了什么
- 代码仓库（未提交，git 由 Jerry 做）：
  - `docs/grokbot_x_spec.md` v1.3：§3.2 名单文件格式、§3.3 白名单和黑名单规则、§3.4 采集节奏和覆盖时长、`response_trailer`、§4.1 已知限制更新、验收清单
  - 新增 `infra/local/config/x_lists/blacklist.yml` v1：8 个账号、8 个关键词，`user_id` 先空着，由 grokbot 查到后补上
  - `infra/local/contracts/brz/x_raw_search.yml`：description 补充信封格式、trailer、缺字段、覆盖时长归一化的说明
- 评审结论：黑名单方向合理，但要以 `user_id` 为主键、带原因代码、关键词只用短语、只有一部分账号进 query；对付成批换号的团伙要靠 Silver

## 决定
- 3 条，已同步到 [[Decisions]]：黑名单 v1、spec v1.3、Silver 按覆盖时长归一化

## 没做完 / 下一步
- grokbot：用 `get_users_by_usernames` 补全黑名单的 `user_id`；确认 `Christophep_a` 的行为证据
- grokbot：重抓一次缺字段的页，查清原因
- grokbot：每日 routine 改成 v1.3，建议每 15 分钟抓 1–2 页
- 白名单 v1（82 个账号）要落成 `infra/local/config/x_lists/whitelist.yml`，要 jaylen 提供带 `user_id` 的名单
- `x_raw_search.yml` 的列还没改，要加 `fetched_ts`、`request_id`、`page_index`、`call_status` 等，下一步做
- Jerry 这台机器的 bash 里没有 `gh`，要装

## 相关
- [[2026-10-14 grokbot X spec]]
- [[Gotchas]]
