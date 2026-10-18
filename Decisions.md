---
tags: [usstockemotions, decisions]
updated: 2026-10-12
---

# Decisions

新决定追加在最下面。仓库里的完整版本见 `docs/data_platform_plan.md` §12。

| 日期 | 决定 | 原因 |
|---|---|---|
| 2026-10-10 | Phase 1 不用 Kafka | 轮询 API，量小，4h bucket，批处理够用。Sink 接口留好，以后能换成 Kafka |
| 2026-10-10 | Bronze→Silver 用 PySpark，Silver→Gold 用 dbt-spark | 复杂解析和 as-of join 用 Spark；建模、测试、合约、血缘用 dbt |
| 2026-10-10 | 计算用 EMR，不用 Databricks | 目标是积累 AWS 经验；表是 Delta 格式，以后还能换 Databricks |
| 2026-10-10 | 版本锁定 Spark 3.5.3 + Delta 3.2.1 | 和 EMR 7.x 对齐，代码不改就能上 AWS。没确认 EMR 支持之前不升 Spark 4 / Delta 4 |
| 2026-10-10 | **本地存储用 RustFS 1.0**（不用 MinIO / SeaweedFS） | MinIO 公开镜像拉不到了；SeaweedFS 删表后留下空目录，Delta 在同一位置重新建表会出错。RustFS 是纯对象语义，和 S3 一样 |
| 2026-10-10 | Metastore 预先导入 Hive 2.3 表结构，关掉 DataNucleus `autoCreateAll` | 自动建表在 Postgres 上第一次 `CREATE TABLE` 时会死锁 |
| 2026-10-10 | 只有一套环境；CI 隔离到 `usse-ci` bucket + `ci_` schema 前缀 | 团队小，够用；以后要 dev 环境可以用 SHALLOW CLONE |
| 2026-10-10 | Bronze 采集排在参考数据之前（M1 在 M2 前） | StockTwits 拿不到历史，每晚一天就少一天数据 |
| 2026-10-10 | 采集是纯 Python，写 JSONL → 再由 Spark append 任务写进 Bronze Delta（推荐方案，M1 PR 里最终确定） | 采集端不需要 JVM，Bronze 照样是 Delta |
| 2026-10-10 | 所有东西放进 `infra/local/`；`.github/` 和 `docs/` 留在根目录 | 基建只是项目的一部分；GitHub 只认根目录的 workflow |
| 2026-10-10 | Bronze/Silver 的 schema 写在合约 YAML 里，DDL 由合约生成（Rule 17）；Gold 用 dbt contract | 只有一个 schema 来源 |
| 2026-10-10 | `spark.sql.sources.default=delta`，同时规定显式写 `USING DELTA`（Rule 16） | 开了 Hive 支持后，漏写 USING 会悄悄建成 Hive 表 |
| 2026-10-10 | 按人和文件夹分模块（Rule 18、19） | 见 [[Team & Ownership]] |
| 2026-10-10 | JupyterLab 装在 spark 容器里，不装在宿主机 | Windows 上直接跑 PySpark 要 winutils/hadoop.dll，很脆弱；本机只有 Java 8；容器里和正式任务的配置完全一致 |
| 2026-10-10 | Notebook 只用来探索；正式代码是带测试的 `.py`；提交前清空输出（Rule 20） | |
| 2026-10-10 | X 数据只通过 Grok `x_search` 给 agent 当工具用 | 不是原始数据，不完整，不可复现 |
| 2026-10-10 | 不用小红书 | 没有 API，爬取违反 ToS，还有隐私法风险 |
| 2026-10-12 | Claude 的记忆统一放进 Obsidian vault `D:\Obisidian_notes\JC` | Jerry 的要求；这台电脑上所有 Claude Code session 共用 |
| 2026-10-12 | **表命名改成 `layer.source_purpose`**：`brz.stocktwits_raw`、`brz.xhs_raw_search`、`brz.x_raw_search`（取代 plan §4.2 的 `raw_` 前缀写法） | Jerry 定的；按来源归组，一眼看出数据从哪来。repo 已同步（plan v1.2） |
| 2026-10-12 | **X 原始帖子纳入 Phase 1**，由 grokbot 拉取（X API v2 结构），取代 2026-10-10 "X 只给 agent 当工具"的决定 | Jerry 定的；grokbot 返回的是原始帖子 + 作者信息，不再是 Grok 的总结 |
| 2026-10-12 | 小红书 Bronze 拆两张表：搜索结果和笔记详情分开 | 两个接口粒度不同；搜索结果没有正文和绝对时间，必须再调详情 |
| 2026-10-12 | **Bronze 粒度统一为"一次 API 响应一行"**，`payload` 存整个响应；展开和关联放 Silver | 采集端最简单、每行可追溯到请求；单行几十 KB，不会 OOM；跟风控无关（请求次数不变） |
| 2026-10-12 | X 的 `ingest_ts` = grokbot 文件到我们手里的时间；grokbot 的请求参数暂时没有，`request_*` 列可空 | grokbot 不记录调用时间和参数；用更晚的时间对 P3 是安全的 |
| 2026-10-14 | **X 过滤只能写在 query 里**（服务器端筛掉，不占额度，query 有记录可复现）；grokbot 抓回来后不做任何过滤，spam 打分放 Silver | grokbot 按调用次数计费，1 次 = 1 页 ≤25 条；抓回来再删省不了钱，也没法改阈值重算 |
| 2026-10-14 | X 分三路：`kol`（白名单原帖）、`kol_replies`（`to:` 评论区，加 `-has:links`）、`crowd`（cashtag，主信号，不能被白名单替代） | 白名单是按今天的名气挑的，有前视偏差，所以要带 `added_on`/`removed_on` 做版本管理；`-has:links` 会把图表截图一起筛掉，只用在评论区 |
| 2026-10-14 | grokbot 信封按它的实际能力降级（spec v1.2）：`response` 保留 grokbot 包过一层的原样，用 `response_format` 标记，多出来的字段到 Silver 再去掉；`call_status`/`error_message` 替代 `http_status`，`quota` 替代 `rate_limit`；时间取调用前后 | grokbot 走的是工具层，拿不到原始 HTTP 响应、状态码和响应头；时间误差 <1 秒，而且偏晚，对 PIT 是安全的 |
| 2026-10-12 | 采集落地目录从 `<lake>/brz/raw_<source>/` 改成 `<lake>/landing/<table>/` | 原路径和 Delta 表目录重叠，VACUUM 会删掉未被 log 记录的 JSONL |
| 2026-10-15 | **X 黑名单 v1**：8 个账号，原因代码分 `scam` / `promo` / `bot`，以 `user_id` 为主键；关键词只用短语（`"check DM"`、`"Contact me"`、`"WhatsApp group"`、`"join my WhatsApp"`）。文件放代码仓库 `infra/local/config/x_lists/blacklist.yml` | 出现次数多不算证据，要有行为证据；spam 号经常改名，只认 username 会失效；单词 `"WhatsApp"` 会误杀 $META 新闻；query 上限 1024 字符，只有 `in_query: true` 的账号进 query，其余交给 Silver；`stock<Name><数字>` 这类成批换号的团伙交给 Silver 的正则规则 (jerry) |
| 2026-10-15 | grokbot spec 升到 v1.3：新增 `response_trailer`；`params` 和 `endpoint` 按实际发送的名字记录；`fetched_at` 容忍度 10 秒；query 上限 1024；建议每 15 分钟抓 1–2 页 | 都来自 jaylen 2026-10-15 的试跑（见 [[2026-10-15 jaylen grokbot X trial run]]） (jerry) |
| 2026-10-15 | **翻页链没翻完时，Silver 的发帖量按"每分钟帖子数"算**，覆盖时长 = 链上最老到最新一条帖子的 `created_at` 之差；缺 `entities` 的页，cashtag 退回正则匹配 | 有页数上限时，越热的票覆盖的时间越短，原始条数会低估热度；试跑里 25 页有 7 页整页缺 `entities` (jerry) |
| 2026-10-18 | **grokbot 输出接口冻结**（spec v1.4 §0）：envelope、manifest、文件命名、时间戳、`query_id` 前缀、名单文件格式都不能改，加字段也不行；要改只能向代码仓库提 PR 改 spec，由 jerry 批准并升版本号。query、名单内容、页数上限、轮询间隔、prompt 措辞可以自由改。摘要见 [[X Collector]] | trial 2 检查 0 FAIL，格式可用；冻结之后 jaylen 改 prompt 和 jerry 写 loader / Silver 可以同时进行，互不阻塞 (jerry) |
| 2026-10-18 | **翻页遇到短页就停**：一页不满 `max_results`（25）就结束，记 `exhausted: true`，哪怕带着 `next_token` | grokbot 在短页上也返回 `next_token`；trial 2 里短页之后的 13 页有 12 页是空的，1 页只有 1 条，省调用、几乎不丢数据 (jerry) |
| 2026-10-18 | **crowd 流在 Silver 必须按 cashtag 过滤**：正文或 `entities.cashtags` 里有 query 中的代码才保留，发帖量在过滤之后再算；query 里照样写 `$` | grokbot 的工具会吞掉 `$`，trial 2 里 `$SPY OR $QQQ` 有 82% 的帖子没有 cashtag；grokbot 不能在抓取后过滤（spec §2） (jerry) |
| 2026-10-18 | **每天调用预算**（spec §3.5）：开盘时段（工作日 13:30–20:00 UTC）`kol_replies` 和 `crowd` 每 15 分钟 2 页，其余时段 1 页；`kol` 始终每 15 分钟 1 页；剩余低于 50 次就暂停到重置 | 配额每天 1000 次；trial 2 每轮 14 次，每 15 分钟一轮要 1344 次；按新分配约 800 次。kol 最便宜、唯一能全量覆盖，最后才砍 (jerry) |
| 2026-10-18 | **X 黑名单 v2**：8 个账号的 `user_id` 补全，`Christophep_a` 确认保留；关键词加 `FLAI`，作为单词关键词的例外（只被垃圾使用的品牌名）；关键词同时用在 `kol_replies` 和 `crowd` | FLAI 色情 AI 女友团伙在 youngstockuser 评论区刷了 25 条；这些号注册早、有粉丝，按账号特征筛不掉；成批换号的团伙交给 Silver 按相同简介识别 (jerry) |
| 2026-10-18 | **所有 Bronze 表的输入统一为 envelope v1**（和 grokbot 同一套格式，见 `docs/grokbot_x_spec.md` §4）：一行 = 一次调用；envelope 字段成为共有列；**失败调用也存**；**PIT 用 `fetched_ts`**，`ingest_ts` 只是 loader 写入时间。plan 升 v1.3 | 一个 loader 就能处理所有数据源；有请求参数和抓取时间才能回溯血缘、做 point-in-time；没有失败记录就分不清"没人发帖"和"采集挂了"；2026-10-18 模拟采集验证过 StockTwits 和小红书都能产出这个格式 (jerry) |
| 2026-10-18 | **`slv.posts` 合约定稿**：PIT 用 `available_ts`（最早一次 `fetched_ts`）；互动数只存最新快照、不满足 PIT；X `includes.tweets` 不算帖子；小红书只有拉过详情的笔记才算；作者只存加盐 hash（`USSE_AUTHOR_SALT`，三人共用）；`lang` 只用来源自带的；`matched_query_ids` 保留所有来源 query；`is_cashtag_verified` 标记 crowd 帖子是否真有 cashtag；用 `first_request_id` 而不是抓取次数 | Bronze 已经把 `fetched_ts` 定为 PIT；快照数会随时间变；引用原帖不是 query 搜到的，算进来会虚增热度；不存原始作者 id；所有合并规则必须幂等（P6），次数在重跑时会累加 (jerry) |
| 2026-10-18 | **垃圾识别现在只用规则（`spam_v1`），不上 ML 模型**；以后要上，放在旁路表 `slv.post_quality_scores`（按 `model_version`），不插在 brz 和 slv 之间 | 没有标注数据；现在的垃圾主要是团伙，按账号群体的规则更准、更好解释；插在中间会破坏 P4 可重建和 P5 版本化，还有前视偏差。M5 让 LLM 顺带输出 `is_spam`，攒标注 (jerry) |
| 2026-10-18 | 基建进度用 [[Phases & Milestones]] 里的 Mermaid 甘特图；需要任务看板时再开 GitHub Projects | Obsidian 和 GitHub 都能直接渲染，三人不用装新工具；跟记忆库一起更新，不会和文字记录对不上 (jerry) |

## 提议了但还没拍板
- 2026-10-14 grokbot 输出规范 `docs/grokbot_x_spec.md`（信封 + `fetched_at` + 请求参数 + 补字段 + 空/失败页也输出 + since_id 翻页到底）。交给 grokbot 后，`x_raw_search.yml` 要跟着改（加 `fetched_ts`/`request_id`/`page_index`/`http_status`，`request_*` 改不可空，PIT 用 `fetched_ts`）
- `user_tag_v0`：用 StockTwits 用户自己打的看涨/看跌标签做临时 label 版本，让 M4 薄切片先跑起来（Q9）
- 采集放在常开的小 VM 上（EC2 t4g.small / Lightsail），直接写真实 S3（Q8）
- 拿到 GitHub 用户名后加 `.github/CODEOWNERS`
- 加 nbstripout git hook，提交时自动清空 notebook 输出
