---
tags: [session, usstockemotions, grokbot, x]
author: jerry
date: 2026-10-18
updated: 2026-10-18
---

# 2026-10-18 第二轮 grokbot 试跑的结果

## 要求
- 看 jerry 自己的 grokbot 跑完的 trial 2（3 轮），找出问题
- prompt 和 spec 按建议改，但改完不再实测；prompt 后续交给 jaylen
- 把问题和要改的 prompt 写进记忆。前提是 schema 冻结：jaylen 的 grokbot 读完记忆之后不能改 schema，这样两边可以同时开发

## 试跑概况
- 跑在 2026-10-18 03:44–03:59 UTC，也就是美东周六晚上 11 点多，全周最冷清的时段。3 轮的间隔实际是 10 分钟和 3 分钟，不是计划的 15 分钟
- 共 45 次调用：42 次搜索（每轮 14 次）、1 次查用户 id、1 次 `get_users_posts`、1 次 refetch。当天配额最后剩 955/1000
- 输出文件 6 个，在代码仓库 `infra/local/data/landing/x_raw_search/trial2/`（本地，不提交）
- grokbot 能把文件直接写进这个文件夹，不用再手动下载附件

## 管道合格的部分
- 检查脚本 **0 FAIL**。43 个 envelope 全部 `call_status: ok`，每个都带配额行，参数名和 endpoint 都按实际发送的记录（`post.fields`、`pagination_token`、`grokbot:search_posts_all`）
- **跨轮衔接对得上**：7 个 query 在 run 2 和 run 3 的 `since_id` 都等于上一轮的 `newest_id`
- 489 条唯一帖子，13 条重复全是跨流的（同一条帖子既是 kol_replies 又带 cashtag），同一个 query 跨轮零重复。Silver 按 post id 去重，同时保留它属于哪些流
- 黑名单 8 个账号一条都没漏进来

## 发现的问题
1. **`$` 被 grokbot 的工具吞掉了**，cashtag 搜索变成了普通单词搜索。`$SPY OR $QQQ` 抓到的帖子 82% 正文里没有 cashtag（"spy game"、"I SPY"、盗号广告），`$NVDA OR $TSLA OR $AAPL` 是 45%。grokbot 的报告也确认了这一点
2. **FLAI 色情 AI 女友团伙**：6 个账号（Camedsujillo、lor288Jaysdxay、lapCriscscDun、ucdLdgoncsfL、yanPagetcs、gcsfVuDou）专门在白名单博主 youngstockuser 下面发广告，内容里混着行情词，结尾都是"快上 FLAI APP"。一共 25 条，youngstockuser 收到的 26 条回复几乎全是它们。这些号注册于 2012–2014 年，有一两千粉丝，所以按账号年龄或粉丝数都筛不掉。它们的简介几乎一样
3. **配额不够**：每轮 14 次，每 15 分钟一轮，一天要 1344 次，超过 1000 次上限。何况这次是最冷清的时段
4. **短页也返回 `next_token`**：13 次翻页拿到的是空页或几乎空页（12 页空，1 页只有 1 条）
5. **整页缺字段更严重了**：32 页里有 14 页没有 `lang`、`entities`、`conversation_id`，38% 的帖子没有 `lang`。refetch 没把字段要回来，说明是工具层面的固定行为
6. **配额行末尾带着 grokbot 运行环境的标签** `</cursor_untrusted_data_1337>`
7. `requested_at` 最多比实际调用早约 14 秒（是 SHOULD 字段，不影响 PIT）
8. kol 这一路每轮只有 1–4 条，很多跟股市无关（橄榄球、中期选举），交给 Silver 做相关性过滤
9. grokbot 报告的其他疑似垃圾：盗号服务（sital_0708、Jerry9611、Halil99513270）、换了措辞的 WhatsApp 引流（JukoMarley、FlorianHessique、FlreIlghtsFDN）

## 做了什么（代码仓库，未提交，git 由 Jerry 操作）
- `docs/grokbot_x_spec.md` 升到 **v1.4**，envelope 不变（`envelope_version` 仍是 1）：
  - 新增 §0 冻结接口，写明什么不能改、什么可以自由改（摘要见 [[X Collector]]）
  - §2 短页停止：一页不满 `max_results` 就结束翻页，记 `exhausted: true`
  - §3.1 `$` 被吞，Silver 必须按 cashtag 过滤 crowd；`kol_replies` 也带垃圾关键词
  - §3.3 只被垃圾使用的品牌名可以作为单词关键词（例如 FLAI）
  - §3.5 新增每天调用预算和初步分配
  - §4.1 更新已知问题，并改为"不要 refetch"
  - §6 `exhausted` 的定义加上"遇到短页"
- `infra/local/config/x_lists/blacklist.yml` 升到 **v2**：8 个 `user_id` 全部补上；`Christophep_a` 已确认（反复发美股荐股推广，带 "Add me on WhatsApp" 和同一个 t.co 链接）；关键词加 `FLAI`
- `docs/grokbot_x_trial_prompt.md` 升到 **v1.4**（未实测）：短页停止，用黑名单 v2，去掉 step 0 和 refetch，`kol_replies` 带关键词（最长的 query 约 970 字符），说明 `$` 会被吞
- `infra/local/scripts/check_grokbot_x.py`：判断"翻完"时把短页也算上。trial 2 的 manifest 是按旧规则写的，现在只报 WARN 不报 FAIL；模拟数据里埋的 8 个错误照样全部抓到

## 决定
- 见 [[Decisions]] 2026-10-18 的几条：接口冻结、短页停止、crowd 的 cashtag 过滤、每天调用预算、黑名单 v2

## 没做完 / 下一步
- jaylen 接手 prompt，任务清单见 [[X Collector]]
- 开放问题：youngstockuser 的评论区还有没有价值（Q14）；白名单要不要加 `kind: news | kol`（Q15）
- `x_raw_search.yml` 的列还没改（`fetched_ts`、`request_id`、`query_id`、`page_index`、`call_status`、`response_format`、`response_trailer`），放在"完善 brz"那一步做
- 接下来：模拟抓一轮小红书和 StockTwits，然后完善 Bronze，再开始 Silver
- 这个分支还没 push，PR 还没开（`gh` 还没装）

## 相关
- [[X Collector]]
- [[2026-10-17 jerry grokbot trial 2 prep]]
- [[2026-10-15 jaylen grokbot X trial run]]
