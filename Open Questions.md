---
tags: [usstockemotions, open-questions]
updated: 2026-10-18
---

# Open Questions

按 Rule 12，标成 **OPEN** 的东西没有 Jerry 确认不能实现。有结论后把状态改掉，并在 [[Decisions]] 里追加一行。

| # | 问题 | 负责人 | 卡住哪个里程碑 | 状态 |
|---|---|---|---|---|
| Q1 | Score 的含义：方向、风险还是关注度（或者多个 score） | Jaylen + Linan | M6 | OPEN |
| Q2 | 股票池（半导体？S&P 100？） | Linan + Jaylen | M1（可先用临时名单）、M2 | OPEN，**现在就要** |
| Q3 | 行业分类：GICS 一级还是自定义 | Jerry + Jaylen | M2 | OPEN |
| Q4 | 4h bucket 怎么对齐、用哪个时区（提议：美东时间从午夜开始切） | Jerry + Linan | M4 | OPEN |
| Q5 | `post_labels.topic` 的 topic 分类体系 | Jaylen | M5 | OPEN |
| Q6 | 小时级价格的数据供应商 | Jerry | M6（M3 期间开始评估） | OPEN |
| Q7 | 采集哪些 subreddit | Jaylen | M1 Reddit | OPEN，**现在就要** |
| Q8 | 常开采集主机用的 AWS 账号和预算 | Jerry | M1 | OPEN |
| Q9 | 临时 label 版本 `user_tag_v0` | Jerry + Linan | M4 | OPEN |
| — | StockTwits API 能不能用 | Jerry | M1 | ✅ 2026-10-12 验证可用（要带浏览器 UA）；Reddit 还没验证 |
| Q10 | StockTwits 要不要回填约 10 个月的历史？回填数据的 likes 等可变字段有 look-ahead 风险 | Jerry + Linan | M1 | OPEN（2026-10-12） |
| — | 小红书能不能抓 | Jerry | M1 | ✅ 2026-10-12 验证可用（用 cookie 导入；扫码登录会被拒） |
| Q11 | 小红书要不要正式纳入 Phase 1？工具不能商用，依赖个人账号 cookie，有封号风险，内容很多是 A 股 / 中文语境 | Jerry + 团队 | M1 | OPEN（2026-10-12） |
| Q12 | 小红书关键词表：按股票池自动生成（中文名 + ticker + "股票"），还是手工维护 | Jerry + Jaylen | M1 | OPEN（2026-10-12） |
| Q13 | 小红书评论要不要采？量大，但情绪信息多 | Jerry + Linan | M1 | OPEN（2026-10-12） |
| Q14 | youngstockuser 的评论区几乎被 FLAI 垃圾团伙占领（trial 2 里 26 条回复有 25 条），它还留在 `kol_replies` 里吗？ | Jaylen | M1 X | OPEN（jerry 2026-10-18） |
| Q15 | 白名单混着新闻和资讯号（MarketWatch、zerohedge、WatcherGuru、DeItaone、unusual_whales 等），要不要加 `kind: news \| kol` 分开统计？加字段属于名单格式变更，要按 [[X Collector]] 走 PR | Jaylen | M3 | OPEN（jerry 2026-10-18） |
