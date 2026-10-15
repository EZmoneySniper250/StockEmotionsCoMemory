---
tags: [session, usstockemotions, grokbot, x]
author: jaylen
date: 2026-10-15
updated: 2026-10-15
---

# 2026-10-15 grokbot X 试跑（spec v1.2）

## 要求
- 按 grokbot X 采集规范 v1.2 和试跑说明 v1.2 跑一次试跑（见 [[2026-10-14 grokbot X spec]]）：2 小时窗口，最多 30 次调用，每个 query 最多 5 页，只抓原始数据，不做抓取后过滤
- 白名单用 @JaylenLinz 关注的股市/投资博主
- 输出 JSONL、manifest、兼容性报告

## 做了什么
- 运行时间：2026-10-14 23:36–23:38 美东（`run_id` = `20261015T033601Z-trial`，`collector_version` = `2026.10.14-1`）
- 共调用 26 / 30 次（含 1 次 `get_users_by_usernames` 查白名单 user_id，不产出 envelope），0 错误，0 超时
- 25 个 envelope（含 2 个 since_id 复查空页），共 527 条帖子；manifest `status: partial`
- 白名单 v1 = 82 个账号（@JaylenLinz 关注的投资类博主）；黑名单 v1 为空（本次只提建议，不过滤）

| query_id | pages | posts | exhausted |
|---|---|---|---|
| kol-q01a | 4 | 54 | true |
| kol-q01b | 3 | 27 | true |
| kol_replies-q01a | 3 | 75 | false |
| kol_replies-q01b | 3 | 74 | false |
| kol_replies-q01a-nolinkfilter | 1 | 25 | false |
| kol_replies-q01b-nolinkfilter | 1 | 24 | false |
| crowd-q01 | 5 | 124 | false |
| crowd-q02 | 5 | 124 | false |

（kol-q01a/b 的页数含最后的 since_id 空页）

### 兼容性发现
- **query 长度**：超过 512 字符的 query 被接受，上限 1024；82 个账号的 `from:` 拆成 a/b 两段
- **`to:`、`-is:nullcast`** 都能用，不需要备选写法
- **`-has:links`**：与 nolinkfilter 对照的重叠部分只多去掉 2 条引流链接，没看到图表截图被误删（样本小，结论不稳）
- **整页缺字段**：25 页里有 7 页（第 1、3、4、9、11、21、22 页）整页没有 `lang`、`conversation_id`、`entities`、`possibly_sensitive`、`context_annotations`；其余页齐全。user 字段全部都有。`entities` 用于 cashtag 对应，影响较大，原因未知
- 多出字段：`article`、`withheld`
- **wrapper 改动**：每条帖子多一个 `url` 字段；JSON 后跟一行非 JSON 的额度文本 → 原文存进 `notes`，数值解析进 `quota`
- **与规格的偏差**：
  - 字段参数实际名为 `post.fields`，记录里按规格写 `tweet.fields`
  - 分页 token 为 `next_token`；endpoint 标签为工具名而非 X 原始路径
  - `fetched_at` 误差 1–6 秒（规格要求 ≤1 秒）
  - 为守 30 次预算，kol_replies 每段只翻 3 页（上限 5），2 小时窗口只覆盖约 4–21 分钟

## 决定
- 无新决定（黑名单待 Jerry 定）

## 没做完 / 下一步
- 建议黑名单账号（仅提议）：`stockRichard11`（crowd 流出现 12 次）、`stockSophia1`、`stockCharlotte7`、`Christophep_a`、`Jeremy_219`（WhatsApp 荐股群）、`CoachDJLW`（关注列表引流）、`stockbull_app`（自动信号机器人）
- 建议黑名单关键词：`"WhatsApp"`、`"Contact me"`、`"check DM"`
- 查整页缺字段的原因；确认后再定是否需要补抓
- 每日 routine 改成 v1.2 格式：原始 envelope 与分析分开存
- 原始 JSONL / manifest 不放这个 repo，由 Jaylen 单独发给 Jerry

## 相关
- [[2026-10-14 grokbot X spec]]
- [[Gotchas]]
- [[Phases & Milestones]]（M1 grokbot X 导入）
- [[Open Questions]]（Q2 股票池决定 crowd 的 cashtag）
