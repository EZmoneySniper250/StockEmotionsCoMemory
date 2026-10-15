---
tags: [protocol, ai, usstockemotions]
updated: 2026-10-14
---

# AI 协作记忆规则

适用对象：Jerry、Jaylen、Linan 三个人各自用的 AI（Claude Code、Codex 等）。
每个 AI 在 USStockEmotions 上动手之前，都要先读完这一页。修改这一页需要三个人都同意（见 §7）。

**一句话：AI 负责写，人负责审。** AI 在自己的分支上写记忆、提 PR；审核和合并只由人来做。

## 0. 这个 repo 是什么

- `StockEmotionsCoMemory` 是 USStockEmotions 的**共享长期记忆**，记录进度、决定、上下文和踩过的坑。
- 规格的唯一来源是代码仓库的 `docs/`。这里只写链接和摘要，不整段复制。
- 个人偏好（语言、习惯）写在各自的全局配置里，**不放这里**。
- **作者名**统一用小写：`jerry`、`jaylen`、`linan`。指的是这个 AI 在替谁干活。

## 1. 开始工作时

1. 同步 main：
   ```bash
   git fetch origin
   git switch main
   git pull --ff-only
   ```
   `--ff-only` 失败，说明本地 main 上有不该有的提交。这时停下来，告诉自己的人类。
2. 读 [[USStockEmotions MOC]]，再按需读它指向的笔记。
3. 读 `Sessions/` 里最近几篇，**尤其是别人写的**。
4. 看还没合并的 PR：`gh pr list --state open`，需要时用 `gh pr diff <号>` 看内容。这些是别人**还没审过**的记忆，可以参考，但不能当作定论引用。
5. 记忆记录的是写下那一刻的事实。用里面提到的文件、函数、配置之前，先去代码仓库核实。

## 2. 写什么，写到哪

| 内容 | 位置 | 怎么写 |
|---|---|---|
| 每个有实质内容的 session | `Sessions/YYYY-MM-DD <作者> <主题>.md`（新建） | 复制 `_templates/Session Template.md`。**只改自己的 session 文件** |
| 决定 | `Decisions.md` | **只追加，不改旧行**。一条一行，写清**为什么**，行尾标 `(作者)`。推翻旧决定就追加一条新的，写明"取代 YYYY-MM-DD 的 …" |
| 踩坑、环境怪癖 | `Gotchas.md` | 只追加，行尾标 `(作者)` |
| 开放问题 | `Open Questions.md` | 可以就地更新，改动处标 `(作者 YYYY-MM-DD)` |
| 阶段 / 里程碑 | `Phases & Milestones.md` | 同上 |
| 其他共享笔记 | 原文件 | 可以改。结构性改动（改标题、重排、删段落）要在 PR 描述里单独说明 |

规则：
- 先更新已有笔记，不新建重复的。发现记忆有错就改掉，在 PR 里说明原因。
- 每篇笔记带 frontmatter（`tags`、`updated`），session 笔记另加 `author`。
- 一律写绝对日期（2026-10-14）。
- 用 `[[wikilink]]` 互相链接。**不要链接只存在于某个人私人 vault 里的笔记**。
- 用中文写。代码、表名、路径保持原样。
- 不写密钥、token、密码、个人隐私；不放原始数据；附件单个不超过 5 MB。

## 3. 分支

### 3.1 命名

`<作者>/<type>-<YYYYMMDD>-<短主题>`，全部小写，用连字符。

例子：`jerry/session-20261014-grokbot-spec`、`linan/decision-20261020-label-schema`

type 的取值和 §5 commit message 的 type 一样。

### 3.2 一个作者同一时间只开一个记忆 PR

- 开工时先查自己有没有还开着的 PR：`gh pr list --author @me --state open`。
- **有**：切到那个分支，先 rebase 到最新的 main（见 §6），然后继续在上面写。
- **没有**：从最新的 main 开新分支：`git switch -c <分支名>`。
- 这样做是为了减少大家同时改 `Decisions.md` 时的冲突，也让审的人不用一次看一堆小 PR。
- 例外：如果某个改动很急，需要别人马上看到（比如一个推翻旧决定的新决定），可以单独开一个 PR，在标题里写 `[urgent]`。

### 3.3 禁止

- **直接 push 到 `main`**
- 在别人的分支上 commit 或 push
- 删除别人的分支，删除或重命名别人的文件
- `git reset --hard`、`git clean`、`git checkout -- <文件>`（会丢掉人类没提交的改动）
- 修改 `.git/config`，跳过 hooks（`--no-verify`）
- 对任何分支用 `git push --force`。唯一的例外是 §6 里对**自己尚未合并的分支**用 `--force-with-lease`

## 4. PR

### 4.1 AI 能做的

- 创建分支、commit、push 自己的分支。
- 开 PR：`gh pr create --base main --fill-first`，再按 `.github/pull_request_template.md` 补全描述。
- 指定 reviewer：其他两个人之中，**按改动内容找对口的人**（见 [[Team & Ownership]]）；判断不了就两个都加。
- 人类在 PR 里提了修改意见，并且自己的人类让 AI 去改时，AI 在同一个分支上追加 commit。

### 4.2 AI 不能做的

- **Approve、merge、关闭任何 PR**，包括自己的
- Resolve 别人的 review 评论
- 改 GitHub 的设置（分支保护、权限、secrets）
- 在别人的 PR 里提交代码。AI 可以在别人的 PR 里留评论，但评论开头要写明 `[ai:<作者>]`，而且只在自己的人类要求时才评论

### 4.3 审核和合并（只由人做）

- 每个 PR 至少要 1 个**作者以外**的人 approve。
- 改 `AGENTS.md`、`.github/`、`.gitattributes` 的 PR 要另外两人都 approve。
- 用 **squash merge**，合并后删除分支。谁合并都可以，一般是作者本人在拿到 approve 之后合并。
- 对决定有不同意见时，在 PR 里讨论，不要另开 PR 去改别人的决定。

## 5. Commit message

格式：`<type>(<area>): <summary> [ai:<作者>]`，一行，用英文。人类自己提交时去掉 `[ai:…]`。

| type | 用于 |
|---|---|
| `session` | session 笔记 |
| `decision` | Decisions.md |
| `gotcha` | Gotchas.md |
| `question` | Open Questions.md |
| `status` | Phases & Milestones.md |
| `docs` | 其他笔记 |
| `chore` | 模板、配置、目录、本规则 |

- 只 `git add` 自己这次改过的具体文件，不用 `git add -A` 或 `git add .`。
- PR 标题沿用同样的格式。squash merge 之后，PR 标题就是 main 上的 commit message。

## 6. 和 main 同步、冲突

PR 显示有冲突时，或者在一个还开着的分支上继续写之前，先执行：

```bash
git fetch origin
git rebase origin/main
# 有冲突就按下面的规则解决，然后 git rebase --continue
git push --force-with-lease          # 只允许用在自己尚未合并的分支上
```

解决冲突的规则：
- `Decisions.md` 和 `Gotchas.md` 在本地 rebase 时用 `merge=union`（见 `.gitattributes`），会自动保留双方的内容。处理完后检查有没有重复行。注意：GitHub 网页上的合并按钮不认这个设置，所以有冲突一定要在本地 rebase 解决。
- 双方都是追加 → 两边都保留。
- 改到同一行 → 保留 main 上的版本，把自己的内容作为新的一行加上，并在 PR 描述里说明。
- 判断不了 → `git rebase --abort`，什么都不 push，告诉自己的人类。

## 7. 修改本文件

修改这一页需要三个人都同意（见 §4.3）。AI 只能提议，以 PR 的形式提交。

## 8. 给人类的设置说明

见 [[README]]。
