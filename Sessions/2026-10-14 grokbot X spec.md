---
tags: [session, usstockemotions]
updated: 2026-10-14
---

# 2026-10-14 grokbot X 采集规范

- 梳理了现有的 `x_raw_search` 合约：只存 grokbot 返回的原始响应，没有请求参数和抓取时间
- 写了 `docs/grokbot_x_spec.md`（v1.1）：每页一个信封，带 `fetched_at` 和请求参数；补齐请求字段；空结果页和失败页也要输出；since_id 增量翻页；三路数据流；白名单和黑名单带版本；每次运行有页数上限
- 写了 `docs/grokbot_x_trial_prompt.md`：试跑，2 小时窗口，最多 30 次调用，每个 query 最多 5 页。多跑 1 页对照，测 `-has:links` 筛掉了多少
- grokbot 自己的回复和我们的思路一致：按调用次数计费，一次 ≤25 条，query 里的过滤不占额度
- 相关决定见 [[Decisions]]
- 待办：Jerry 填白名单和黑名单后跑试跑，拿结果对照 spec §8 的验收清单检查；结果出来再改 `x_raw_search.yml`（加 `fetched_ts`、`request_id`、`page_index`、`http_status` 和三个 list version 列）
- grokbot 回复了能力限制，spec 和 trial 升到 v1.2，按它的实际能力调整（见 [[Gotchas]] 的 X 部分）。trial 加了运算符的备选写法，报告里要逐个列出字段是否出现、包装层加了哪些字段
- trial prompt 由 Jerry + Jaylen 共同负责，窗口改成 2 小时
- repo 改动未提交
