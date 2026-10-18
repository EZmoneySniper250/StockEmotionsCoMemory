---
tags: [readme, usstockemotions]
updated: 2026-10-14
---

# StockEmotionsCoMemory

USStockEmotions 的共享记忆库，Jerry、Jaylen、Linan 三个人和各自的 AI 共用。AI 的读写和 git 规则见 [[AGENTS]]。

**流程：AI 在自己的分支上写，提 PR；我们三个人互相审，审过了再由人来合并。** `main` 只能通过 PR 修改。

## 第一次设置（每人做一次）

1. **Clone 到自己的 Obsidian vault 里**（路径随意），例如：
   ```bash
   cd "<你的 vault>"
   git clone https://github.com/EZmoneySniper250/StockEmotionsCoMemory.git USStockEmotions
   ```
2. **让 AI 知道这个记忆库在哪。** 把下面这段加进自己的全局 AI 配置：Claude Code 是 `~/.claude/CLAUDE.md`，Codex 是 `~/.codex/AGENTS.md`。把路径和名字换成你自己的：
   ```markdown
   ## USStockEmotions shared memory
   Shared project memory for USStockEmotions lives in `<path to your clone>`.
   Before doing any work on USStockEmotions, read its `AGENTS.md` and follow it,
   including its branch and PR rules (never push to main; humans review and merge).
   My author name is `<jerry|jaylen|linan>`.
   ```
3. **检查 git 身份**：`git config user.name` 和 `git config user.email` 要是你自己的。AI 的 commit 也用这个身份，再用 `[ai:<名字>]` 标明是 AI 写的。
4. **装 GitHub CLI 并登录**：`gh auth login`。AI 用它来开 PR、看别人的 PR。
5. **仓库权限**（Jerry 做一次）：把 Jaylen 和 Linan 加为 collaborator（Write 权限），然后给 `main` 开分支保护：要求 PR（approve 数设为 0，作者本人看过就能合并）、禁止 force push、要求线性历史，并且只允许 squash merge。

## 平时

- 自己手动改笔记也走分支和 PR，流程和 AI 一样（[[AGENTS]] §3–§6），只是 commit message 里不加 `[ai:…]`。
- 审 PR 时重点看三件事：决定有没有写清楚原因；有没有改别人的旧条目；有没有夹带敏感信息。
- 如果用 Obsidian Git 插件：只开"打开时自动 pull"，**不要开自动 commit 和自动 push**。分支保护开了以后，自动 push 到 `main` 本来也会被拒。
- 平时看笔记保持在 `main` 上，就能看到已经审过的版本。
- `.obsidian/` 不进 repo，每个人的 Obsidian 设置互不影响。
