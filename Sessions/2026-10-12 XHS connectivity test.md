---
tags: [session]
project: USStockEmotions
date: 2026-10-12
---

# 2026-10-12 XHS (小红书) connectivity test

## 用户要求
- 用 `D:\xhs_all_in_one\XHS_ALL_IN_ONE`（cv-cat 开源项目，**仅学习用途，非商用**）连小红书，测多关键词搜索，偏金融方向
- 测完汇报；之后定 StockTwits + 小红书的 schema，再想 Silver

## 做了什么
- 本机全局 Python 起后端：`python main.py --host 127.0.0.1 --port 8000`（未起前端，frontend 没 npm install）
- 本地平台用户 `usse_local`（只在该项目的 SQLite 里）
- 扫码登录失败（最终换 `web_session` 时小红书返回"登录异常"）→ 改用浏览器 cookie 导入，成功（account_id=1）。cookie 文件在 `D:\xhs_all_in_one\xhs_cookie.txt.txt`，不要入库、不要提交
- 11 个关键词 × 2 页（按最新排序），22 次请求全部 200，间隔 3–6 s，没被风控
- 详情接口、评论接口各测 1 次，都 OK
- 细节见 [[Gotchas]] 的"小红书"一节

## 结论
- 能用。偏金融的词（`英伟达股票`、`芯片股`、`半导体股票`、`NVDA`、`美股 英伟达`）相关度明显高于宽泛词（`AI`、`半导体`、`英伟达` 混进大量招聘/学习/二手）
- 多关键词：接口一次一个词，多词就循环；空格拼接（`美股 英伟达`）有效，相当于收窄
- 11 个词 22 页 = 429 条，去重后 394 条；同主题不同词之间重叠很小（≤5/40）→ 关键词集合决定覆盖面，需要去重

## 同一个 session 的后续：Bronze schema
- Jerry 定了命名 `layer.source_purpose`，X（grokbot 拉取，X API v2 原始响应）加入 Phase 1。见 [[Decisions]]
- Bronze 统一为一次 API 响应一行；Jerry 问过风控和 OOM：都没影响（请求次数不变，单行几十 KB），真正要注意的是小文件（OPTIMIZE）和采集端逐页写出
- Silver 的问题（编辑链去重、cashtag 正则、垃圾帖规则、X 抽样）Jerry 说**先不考虑**，只记在 [[Gotchas]]
- repo 改动（未提交）：
  - 合约：`brz/stocktwits_raw.yml`（由 `raw_stocktwits.yml` 改名重写）、`xhs_raw_search.yml`、`xhs_raw_note.yml`、`x_raw_search.yml`；`_template.yml` 示例名
  - `docs/data_platform_plan.md` v1.2：§1.3、§3.3、§4.2、§5、§6、§12、§13（Q10–Q13）
  - `docs/dev_plan.md`：M1 目标和交付物、落地路径改到 `landing/`、表名
  - `docs/architecture.md`：数据流图
  - `docs/img/data_flow.png` 还是旧图，没有重新渲染

## 下一步
- Jerry review 后自己提交
- M1：`tables.py`（读合约生成 DDL）→ `sink.py` → fetchers
