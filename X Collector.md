---
tags: [usstockemotions, grokbot, x, interface]
updated: 2026-10-18
source: docs/grokbot_x_spec.md v1.4 §0
---

# X 采集（grokbot）：冻结的接口和可以改的部分

> 任何人或 AI 改 grokbot 的 prompt 之前，先读这一页。规格的唯一来源是代码仓库的 `docs/grokbot_x_spec.md`（v1.4，§0），这里只是摘要。

## 为什么要冻结

grokbot 产出的文件格式是采集端和我们导入程序（Bronze loader）之间的合同。格式冻结之后，两边可以同时开发：
- jaylen 那边改 prompt、改 query、调名单，不影响 Jerry 这边写 loader 和 Silver。
- Jerry 这边写 loader 和 Silver，也不需要等 prompt 定稿。

2026-10-18 的第二轮试跑，检查脚本 0 个 FAIL，说明当前格式可以用（见 [[2026-10-18 jerry grokbot trial 2 results]]）。

## 冻结的部分：不能改

要改只能向代码仓库提 PR 改 spec，由 jerry 批准，同时升版本号：

| 内容 | spec 章节 | 要升的版本号 |
|---|---|---|
| envelope 的字段：名字、类型、含义、MUST/SHOULD、什么时候为 null | §4 | `envelope_version`（现在是 1） |
| `response` 和 `response_trailer` 的形状 | §4、§4.1 | `response_format`（现在是 `grokbot_x_v2_wrapped_v1`） |
| manifest 的字段，以及 `exhausted` 的含义 | §6、§2 | `envelope_version` |
| 文件命名、JSONL、先写 `.tmp` 再改名、存放位置 | §6 | `envelope_version` |
| 时间戳格式，以及每个时间戳记的是哪个时刻 | §5 | `envelope_version` |
| `query_id` 必须以流名开头（`kol-`、`kol_replies-`、`crowd-`） | §3 | `envelope_version` |
| 名单文件的字段（`whitelist.yml`、`blacklist.yml`、`watchlist.yml`） | §3.2 | spec 版本 |

**加字段也算修改。** envelope 和 manifest 里一个字段都不能加，可选字段也不行。额外信息一律写进 `notes`。

## 可以自由改的部分

这些改动不用改 spec，因为 envelope 本身会记下实际用了什么：
- query 字符串、名单怎么拆成多个 query、`query_id` 的后缀
- 名单内容：增删账号和关键词，同时把名单文件的 `version` 加一
- 页数上限、轮询间隔、每天的调用分配（在 spec §3.4、§3.5 的范围内）
- prompt 的措辞和顺序、重试逻辑、调度方式

prompt 做不到某条冻结规则时，把实际情况写进 `notes`，在报告里说明。**不能为了迁就 prompt 去改格式。**

## 当前版本（2026-10-18）

| 东西 | 版本 | 位置（代码仓库） |
|---|---|---|
| 规格 | spec v1.4 | `docs/grokbot_x_spec.md` |
| 试跑 prompt | v1.4，未实测，交给 jaylen | `docs/grokbot_x_trial_prompt.md` |
| 白名单 | v1，82 个账号（big 47 / small 35） | `infra/local/config/x_lists/whitelist.yml` |
| 黑名单 | v2，8 个账号（`user_id` 全了），9 个关键词 | `infra/local/config/x_lists/blacklist.yml` |
| 检查脚本 | 按 spec v1.4 | `infra/local/scripts/check_grokbot_x.py` |

## 交给 jaylen 的 prompt 任务（2026-10-18）

jerry 把试跑 prompt 交给 jaylen 继续改，不再由我们实测。每一项都只属于"可以自由改"的范围：

1. **把试跑 prompt 改成正式的定时采集 prompt**：每 15 分钟一轮，用 spec §3.5 的调用分配（开盘时段回复类和 crowd 每轮 2 页，其他时段 1 页；kol 每轮 1 页），每天剩余调用低于 50 次就暂停。grokbot 自己说过可以用定时任务每 15 分钟唤醒它。
2. **短页停止**：一页不满 25 条就不再翻下一页，记为 `exhausted: true`。v1.4 已经写了，要确认 grokbot 真的照做。
3. **垃圾关键词变体**：grokbot 建议补 `whats`、`hack`、`telegram channel`。按 spec §3.3，单词容易误杀正常帖子，每加一个先跑一页对照，确认它过滤掉的是什么，再加进 `blacklist.yml` 并升版本号。
4. **`$` 被吞的问题**：可以问 grokbot 的工具有没有办法保留 `$`（比如转义）。在找到办法之前，query 里照样写 `$`，由 Silver 按 cashtag 过滤，**不要在 prompt 里让 grokbot 自己过滤**（违反 spec §2）。
5. **输出位置**：正式采集时写到哪个文件夹（同一台机器、同步盘，还是以后的 S3），定下来后告诉 jerry。

改完的 prompt 用检查脚本跑一遍：`python infra/local/scripts/check_grokbot_x.py <输出文件夹>`。0 个 FAIL 才算没有破坏接口。

## 相关
- [[Decisions]]（2026-10-18 的几条）
- [[Open Questions]]（Q14、Q15）
- [[2026-10-18 jerry grokbot trial 2 results]]
- [[2026-10-15 jaylen grokbot X trial run]]
