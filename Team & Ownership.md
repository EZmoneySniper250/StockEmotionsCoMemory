---
tags: [usstockemotions, team]
updated: 2026-10-12
---

# Team & Ownership

| 成员 | 负责领域 | 文件夹 |
|---|---|---|
| **Jerry** | 采集、存储、Bronze/Silver/Gold 管道、DQ、基建、CI/CD | `infra/`、`.github/`、`docs/` |
| Linan | 标签 schema（和 Jaylen 一起）、score/alert 规则、模型训练、验证 | `modeling/`、`labeling/schema/`、`labeling/prompts/`、`infra/local/dbt/models/gld/scoring/`、`.../alerting/` |
| Jaylen | Topic 分类体系、LLM 标签抽检、投资假设、结果复盘 | `research/`、`labeling/taxonomy/`、`labeling/spot_checks/` |

交叉的地方：
- LLM 打标签：规格（prompt、分类体系）放在 `labeling/`，归 Linan/Jaylen；跑批代码放在 `infra/local/src/usse/labeling/`，归 Jerry
- grokbot X 试跑 prompt（`docs/grokbot_x_trial_prompt.md`）：Jerry + Jaylen 共同负责（2026-10-14）
- Score/alert：规则说明放在 `modeling/rules/`；SQL 放在 `gld/scoring`、`gld/alerting`，归 Linan，Jerry review

**Claude 的工作范围**：替 Jerry 干活时，只动 Jerry 负责的文件夹（Rule 18）。要动别人的文件，就提 PR 请对方 review。
