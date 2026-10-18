---
tags: [usstockemotions, gotchas]
updated: 2026-10-12
---

# Gotchas

## StockTwits API（2026-10-12 实测）
- **必须带浏览器 User-Agent**。curl 默认的 UA 会被 Cloudflare 拦下（403，`Cf-Mitigated: challenge`）；带上 Chrome 的 UA 后，curl 和 Python `requests` 都返回 200
- 不需要认证；响应里没有限流相关的 header（实际限额未知）
- 每页固定 30 条，`limit` 参数没有效果；`max=<id>` 往更早翻，`since=<id>` 取更新的
- 历史大约能回溯 10 个月：`max=640000000` 还能拿到 2025-12-23 的数据，`max=600000000` 返回空
- 一条 message 会出现在它提到的每个 symbol 的 stream 里 → Bronze 里会有重复，Silver 去重

## 小红书 / XHS（2026-10-12 实测，经 XHS_ALL_IN_ONE 本地后端）
- 工具：`D:\xhs_all_in_one\XHS_ALL_IN_ONE`，**仅学习用途**。API 文档 `localhost:8000/docs`；调接口先要本地平台 JWT，再要一个绑定好的小红书 PC 账号（`account_id`）
- **扫码登录会被拒**（最后换 `web_session` 那步返回"登录异常"）→ 改用浏览器 cookie 导入（`POST /api/accounts/import-cookie`），cookie 里要有 `a1`、`webId`、`web_session`。cookie 会过期，要定期更新
- 搜索 `POST /api/xhs/pc/search/notes`：每页约 20 条，`sort_type_choice=1` 表示按最新排序；一次只能传一个关键词，空格拼起来可以做组合查询；返回里混有 `hot_query` 项，要过滤掉
- **搜索结果里没有正文，也没有绝对时间**，只有 `corner_tag_info` 里的"17分钟前 / 昨天 23:12 / 05-22"这类相对时间 → 每条都要再调一次详情 `POST /api/xhs/pc/notes/detail`，才能拿到 `desc`、`time`(ms)、`last_update_time`、`ip_location`、`tag_list`、`interact_info`
- 调详情要带搜索返回的 `xsec_token`（`note_url` 里已经带上了）；token 会过期，最好搜完马上取详情
- 评论 `POST /api/xhs/pc/notes/comments`：返回 `comment_id`、`parent_comment_id`、`created_at_remote`(ms)、`like_count`
- 计数字段都是字符串，而且可能是空串 `''`（不是 0）
- 不同关键词的新鲜度差别很大：`美股` 两页只覆盖约 1 天，`NVDA` 两页能翻回 05-20（就算按最新排序也会混进旧帖）
- Windows 控制台打印中文会乱码，加 `PYTHONIOENCODING=utf-8`

## X（grokbot 拉取，X API v2 search 返回结构；2026-10-12 看样本）
- 一次响应 = `data`（≤25 条帖子）+ `includes.users`（作者去重，一个作者只出现一次）+ `meta`（`result_count`、`newest_id`、`oldest_id`、`next_token`）
- **响应里没有请求参数**（query、start/end time、用的 pagination token），必须由采集端自己记下来，不然 Bronze 丢血缘
- `referenced_tweets`（quoted / replied_to）和 `in_reply_to_user_id` **只在有值时出现**，不是完全没有；`lang`、`conversation_id`、`entities`（cashtags）确实没有
- 编辑过的帖子：同一条编辑链的新旧 id 会**同时**出现在结果里（`edit_history_tweet_ids` 相同）→ Silver 按编辑链去重，取链上最后一个 id
- 垃圾帖样本：新注册账号正文堆 40 个 cashtag；不同账号发几乎相同的回复（"I really like his articles, too @..."）
- `$SPY OR $QQQ` 一页 25 条只覆盖约 33 分钟
- grokbot（2026-10-14 它自己说的）：响应包了一层，每条帖子多一个 `url` 字段，末尾附带剩余额度；拿不到 HTTP 状态码和 `x-rate-limit-*` 响应头；`max_results` 固定 25；`to:`、`-is:nullcast`、`note_tweet`、`context_annotations` 支不支持要等试跑确认

## 环境
- **Git Bash + `docker compose exec`**：前面要加 `MSYS_NO_PATHCONV=1`，不然容器里的路径（`/workspace/...`）会被 MSYS 改坏。
- **MinIO 镜像拉不到了** → 用的是 RustFS 1.0.0（pin 在 2026-09-16 的 GA 版本）。
- **SeaweedFS** 删表后留下空目录，Delta 在同一位置重新 `CREATE TABLE` 会失败。不要换回去。
- **Hive metastore + Postgres**：DataNucleus `autoCreateAll` 第一次 `CREATE TABLE` 时死锁。表结构从 `infra/local/docker/metastore/*.sql` 预先导入，要一直关着。重置方法是删掉 metastore volume（`docker compose down -v` 会清空**所有**本地数据）。
- **Delta 在 S3 上的并发**：Delta 3.2 默认只保证同一个 driver 内的并发写入安全 → **每张表只能有一个写入任务**。AWS 上真要并发，就开 DynamoDB LogStore。
- Delta **只在单表内保证 ACID**，不能跨表 → 每个任务都要能安全重跑（幂等）。
- Delta 没有 PK/FK 约束，只有 `NOT NULL` 和 `CHECK`。主键唯一靠 MERGE + DQ 来保证。
- `VACUUM` 之后，被清掉的旧版本就回溯不到了（默认保留 7 天）。
- Spark UI：4040 端口是 notebook 的会话，同时用 `exec` 跑的任务在 4041。
- JupyterLab 的 token 是 `.env` 里的 `JUPYTER_TOKEN`，端口只绑定本机。
- dbt step 用的是 step 级别的 `working-directory`，相对路径从仓库根目录算，所以 CI 里要写完整路径 `infra/local/dbt`。
- grokbot X 试跑（2026-10-15）：`search_posts_all` 间歇性整页丢字段，25 页里 7 页整页没有 `lang`、`conversation_id`、`entities`、`possibly_sensitive`、`context_annotations`，导入时这些字段要允许为空 (jaylen)
- grokbot X 试跑（2026-10-15）：query 上限 1024 字符（>512 也接受），82 个 `from:` 要拆两段 (jaylen)
- grokbot X 试跑（2026-10-15）：wrapper 在 JSON 后附一行非 JSON 的额度文本，直接 `json.loads` 会失败，要先切掉存进 `notes` (jaylen)
- grokbot X 试跑（2026-10-18）：`search_posts_all` 会吞掉 query 里的 `$`，cashtag 变成普通单词搜索（`$SPY` 搜到 "spy game"），crowd 必须在 Silver 按 cashtag 过滤 (jerry)
- grokbot X 试跑（2026-10-18）：帖子不满一页时也返回 `next_token`，下一页几乎都是空的，一页不满 25 条就停 (jerry)
- grokbot X 试跑（2026-10-18）：配额行后面带着运行环境的标签 `</cursor_untrusted_data_1337>`，loader 用正则只取配额数字 (jerry)
- grokbot X 试跑（2026-10-18）：整页缺字段在 32 页里出现了 14 页，用同样参数 refetch 也补不回来，不要重抓 (jerry)
