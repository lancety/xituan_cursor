---
name: xituan-multirepo-codebase-sync
description: >-
  Syncs the shared xituan_codebase Git submodule across xituan_backend, xituan_cms,
  xituan_platform, xituan_site, and xituan_wechat_app using a fixed order: backend
  codebase first (when not skipped), then four consumer codebases (when each parent main is
  not skipped), then five main repos and xituan_agent. Per main repo: if the main repo has
  nothing to sync (clean, on master, aligned with origin/master), skip that entire project—no
  submodule pull/commit/push for its codebase path and no main-repo pull/commit/push. Stash:
  only pop/apply stashes created in the same run with a unique STASH_TAG; never bare stash pop
  or historical stash@{n}. Every target repo must be on branch master before Git steps; if
  checkout master fails, stop and hand off to the user. Main repos: commit all relevant
  working-tree changes (not submodule pointer only) and push origin master unless the user
  explicitly asked submodule-only bump. Agent runs git only in known paths—no exploratory
  codebase search unless resolving conflicts or git errors. Commit messages written during
  this flow must not include Cursor-related remarks or meta (see skill body). Use when the user asks to push
  codebase submodule, full-repo commit, multirepo sync, or align all projects after shared typing/util changes.
---

# Xituan multirepo: xituan_codebase + main projects + agent

`xituan_module` 根目录**不是**单一 Git 仓库。以下每个目录是**独立**远程仓库，且多数包含子模块 `submodules/xituan_codebase`（与 `xituan_backend/submodules/xituan_codebase` 同源）。

## 仓库清单（5 个主项目 + agent）

| 路径 | 说明 |
|------|------|
| `xituan_backend/submodules/xituan_codebase` | 共享 codebase：**流程起点**，先于此完成 pull / 提交 / 推送 |
| `xituan_backend` | 主项目 + submodule |
| `xituan_cms` | 主项目 + submodule |
| `xituan_platform` | 主项目 + submodule |
| `xituan_site` | 主项目 + submodule |
| `xituan_wechat_app` | 主项目 + submodule |
| `xituan_agent` | 文档/Agent 仓库（无 submodule），**整链最后**提交 |

`.cursor/` 通常不是 Git 仓库。

## AI 执行约束（减少无效检索）

本技能触发时，**默认只按下方阶段在已列出的固定路径里执行 Git 工作流**（终端 `cd` + 本文件给出的命令序列）。目标是把子模块与主仓推到一致状态，**不是**阅读或理解业务代码。

**不要做（除非用户明确要求或进入「意外处理」）：**

- 语义检索 codebase、为「确认目录结构」而大范围 `grep`/读文件、打开与 Git 同步无关的源码。
- 为自拟步骤而探索仓库（路径以本技能表格与阶段标题为准，已写死）。

**允许且鼓励的轻量命令（仍属工作流内）：** 当前仓库下的 `git status`、`git diff`、`git log -1`、`git stash list` 等，用于判断是否有可提交变更、是否干净、stash 是否存在——**仅限当前正在处理的那一个 repo 路径**。

**意外处理（此时 AI 主动介入，可读写冲突文件、补充命令）：**

- Merge / **`stash pop`（仅限「Stash 使用规则」允许的条目）** / `pull` **冲突**：在**出问题的那个仓库目录**内解决（编辑冲突标记、`git add`、完成 merge commit 或按文档「常见边界」drop **本条任务创建且 message 已核对**的 stash 等），然后**回到本技能既定顺序**继续下一阶段。
- **非零退出**、认证失败、detached HEAD、子模块路径缺失等：先用该目录下的 `git status` 与错误输出定位，再采取最小修复；**仍避免**无关代码检索。

**冲突策略（与上衔接）**：任一步出现 merge/rebase 冲突，由 AI 在对应仓库路径内解决后再继续。

---

## `master` 分支与 AI 终止条件（强制）

在进入任一 Git 仓库路径并执行 `pull` / `commit` / `push` **之前**（含：阶段 1–2 的各 `xituan_codebase` 子模块目录、阶段 3 的五个主仓库根目录、阶段 4 的 `xituan_agent`）：

1. **检查当前分支**：`git rev-parse --abbrev-ref HEAD`  
   - 期望输出为 **`master`**。  
   - 若为 **`HEAD`**（detached）或其它分支名：**必须先** `git checkout master`（若有未提交改动阻碍切换，可先按本技能其它条目的 stash 策略处理，再重试 `checkout master`）。
2. **`git checkout master` 失败**（例如分支不存在、冲突、detached 无法安全切回、工作区无法清理等）：**立即终止**与本技能相关的后续所有 Git 操作；向用户报告**仓库绝对路径**、当前 `HEAD` / 分支、完整错误输出，并说明需**手动调整**后用户再发起同步。**不要**改试其它分支、不要强行 `push` 非 `master`。
3. **推送目标**：本技能默认所有 `push` / `pull` 的远程分支为 **`origin master`**。若用户在同一会话中**明确**指定其它分支名，可按用户指令覆盖；否则一律 `master`。

---

## 整 repo 跳过（主仓库无待同步）

**目的**：主项目若当前**不需要**做任何 Git 同步，则**不必**为其 `submodules/xituan_codebase` 执行拉取/提交/推送，也**不必**对该主仓库根目录执行阶段 3 的 pull/commit/push，从而减少无效操作。

### 判定「主仓库无待同步」（对 `xituan_backend`、`xituan_cms`、`xituan_platform`、`xituan_site`、`xituan_wechat_app` 各自主仓库根目录）

在用户**未**明确要求「强制全量同步 / force full sync / 忽略跳过」等前提下，若**同时**满足下列全部条件，则视为该主项目 **整 repo 跳过**：

1. **分支**：已通过本文件「`master` 分支与终止条件」：当前为 `master`（含从 detached/其它分支成功 `checkout master` 后）。
2. **工作区**：`git status --porcelain` 为空（无已修改、已暂存、未跟踪等待处理项）。
3. **与远程一致**：对 `origin` 执行 `git fetch origin master` 后，`HEAD` 与 `origin/master` 指向同一提交（例如 `git rev-parse HEAD` 与 `git rev-parse origin/master` 相同；或 `git status -sb` 中相对 `origin/master` **无** `[ahead …]` / `[behind …]`）。

**则对该主项目**：

- **阶段 1（仅 `xituan_backend`）**：若 **`xituan_backend` 主仓库**满足上述「无待同步」，则**整个阶段 1 跳过**——**不要**进入 `xituan_backend/submodules/xituan_codebase` 做 pull / add / commit / push。
- **阶段 2（四个 consumer 子模块）**：在处理 `xituan_cms/submodules/xituan_codebase` 等路径之前，先对**其父主仓库**（如 `xituan_cms`）根目录做上述判定；若该主项目 **整 repo 跳过**，则**不要**进入对应 `submodules/xituan_codebase`，**不要**对该子模块执行 stash / pull / pop / commit / push。
- **阶段 3（五个主项目）**：若该主项目在阶段 2 已按上条 **整 repo 跳过**，或重新按同一条件判定仍为 **无待同步**，则**跳过**该主仓库根目录下的 pull / stash / commit / push 段落。

**强制全量**：若用户在同一会话中明确要求不跳过（如「强制同步子模块」「force」），则**忽略**本节跳过规则，对该次任务仍按阶段 1–3 的全流程执行。

### 局限（行为说明）

若仅 **`xituan_codebase` 远程**已有新提交，而各主仓库的 `origin/master` **尚未**更新子模块指针、且主仓工作区仍显示「干净」，则按本节会 **跳过** 子模块拉取，**不会**主动把本地嵌套子模块追到最新远程。需要时由人工合并/发版更新主仓指针，或让用户使用「强制全量同步」后再跑本技能。

---

## Stash 使用规则（强制）：仅限**本次 multirepo sync 任务**内创建的条目

本技能里凡提到 stash，**只**允许操作**当前这一次**同步流程中、在**当前正在处理的那个仓库路径**里、用下面规则**新创建**的 stash。**禁止**利用或触碰 `git stash list` 里任何历史条目（例如数月前的 `stash@{1}`、`stash@{2}`），也**禁止**无参 `git stash pop` 后「碰巧」应用到错误条目。

1. **唯一标签 `$STASH_TAG`**：`STASH_TAG="multirepo-sync-$(openssl rand -hex 8)"`（或同等长度随机串）。在**本轮第一次**需要执行 `stash push` 之前生成；其后每个**尚未 pop 的**目录第一次 stash 可使用**当前** `$STASH_TAG`。**同一仓库路径**若在本轮内需第二次 stash，必须**重新生成**新的 `$STASH_TAG`（见规则 6）。每个 stash 的 `-m` message **必须恰好等于**当时生效的 `$STASH_TAG`（可唯一匹配）。
2. **创建**：仅当该目录工作区非空需暂存时执行  
   `git stash push -u -m "$STASH_TAG"`。若输出为 nothing to save / 无新 stash，则记录**本目录本轮未创建 stash**，后续**不得**对该目录执行 `stash pop` / `stash apply`。
3. **恢复**：仅当**本目录**在步骤 2 **确实创建成功**后，在 `pull`（等）之后执行**带标签定位**的恢复，例如：  
   `git stash pop "stash^{/$STASH_TAG}"`  
   （或：`git stash list` 中**仅当存在一条且 message 与 `$STASH_TAG` 完全一致**时，对该条使用显式 ref pop）。  
   **禁止**：裸 `git stash pop`；**禁止**：`git stash pop stash@{0}` 而不先核对 message（`@{0}` 不一定是本轮创建的）；**禁止**：`stash@{1}`、`stash@{2}` 或任意未带 `$STASH_TAG` 校验的条目。
4. **若 `stash pop` / `apply` 失败**（冲突、条目不存在）：**不得**改 pop 其它 stash 条目「碰运气」。在**当前目录**内按「意外处理」解决冲突或中止并交给用户；历史 stash 一律留待用户自行处理。
5. **主仓库与 consumer 子模块**：每个 Git 目录独立一套「是否在本轮创建过 stash」的状态；**禁止**在 A 仓库创建了 stash，却在 B 仓库执行 `stash pop`。
6. **同一仓库路径在本轮内需第二次 stash**（例如 consumer 子模块已 pop 干净后，阶段 3 同一机器上的主仓库再为 `pull` 而 stash）：须再生成**新的** `$STASH_TAG`（全新随机串），再 `push` / `pop` 配对使用。**禁止**在 `stash list` 里仍存在未成功 pop 的**本轮**条目时，再次 `push` 使用**完全相同**的 message（否则 `stash^{/...}` 可能匹配多条）。

---

## 主仓库提交范围（与「仅 bump 子模块」区分）

阶段 3（及用户点名 subset 时的对应主仓库）：

- **默认**：将**主仓库内与本次同步相关的全部工作区变更**与子模块指针**一并**纳入提交：`git add -A`（或用户指令中给出的路径集合）→ `git commit` → **`git push origin master`**。包含业务代码、配置、未跟踪文件等，与子模块 SHA 更新在同一提交或拆成多个提交均可，但**不得**在未经用户说明的情况下故意只提交 `submodules/xituan_codebase` 而丢弃或留空其它已修改的主仓文件。
- **例外**：仅当用户明确写出「只 bump 子模块 / submodule only / 不要提交业务代码」等指令时，才可只 `git add submodules/xituan_codebase` 并单独提交。

若执行 `git add -A` 后仍无变更可提交且已与 `origin/master` 一致，则跳过 `commit` / `push`。

---

## Commit message 约束（强制）

本技能流程内任意步骤产生的 **`git commit`**（阶段 1 `xituan_codebase`、阶段 2 consumer 子模块、阶段 3 主仓、阶段 4 `xituan_agent`）之 **message 正文**：

- **禁止**包含与 Cursor 相关的备注或元信息（例如 `Cursor`、`Co-Authored-By: Cursor`、`Generated by` 类 AI/IDE 署名、点名 `.cursor` 配置或插件等）。
- 仅写与变更本身相关的摘要，风格与团队 conventional commit 或既有习惯一致，**不得在远程可见历史中留下工具/助手痕迹**。

---

## 固定执行顺序（必须遵守）

整体顺序：**① backend 子模块 codebase（可跳过）→ ② 其余 4 个子模块 codebase（按主项目逐个可跳过）→ ③ 5 个主仓库（逐个可跳过）→ ④ xituan_agent**。

### 阶段 1 — Codebase：仅从 `xituan_backend` 子模块开始

目录：`xituan_backend/submodules/xituan_codebase`

**先做「整 repo 跳过」判定**：在 **`xituan_backend` 主仓库根目录**（非子模块内）按上文「主仓库无待同步」检查；若 **整 repo 跳过**，则**不执行**本阶段下文任何命令，直接进入阶段 2。

否则在子模块目录 **先做「`master` 分支与终止条件」检查**；通过后顺序必须是：**pull → add → commit → push**（先与远程对齐，再落本地提交）。

```bash
cd xituan_backend/submodules/xituan_codebase
git rev-parse --abbrev-ref HEAD   # must be master; else checkout master or STOP
git status
git pull origin master
git add -A
git commit -m "feat(scope): concise summary"   # only if there are staged changes
git push origin master
```

记录推送后的 `HEAD` SHA（若本阶段未跳过），便于核对下游子模块是否一致。

---

### 阶段 2 — 其余 4 个项目的 **codebase 子模块**（consumer）

对以下路径**依次**处理（顺序可固定为：`xituan_cms` → `xituan_platform` → `xituan_site` → `xituan_wechat_app`）：

- `xituan_cms/submodules/xituan_codebase`
- `xituan_platform/submodules/xituan_codebase`
- `xituan_site/submodules/xituan_codebase`
- `xituan_wechat_app/submodules/xituan_codebase`

**每个条目开始前**：在**对应主仓库根**（如处理 `xituan_cms/submodules/...` 前先在 `xituan_cms/`）做「整 repo 跳过」判定；若跳过，**本条目不进入子模块**，直接进入下一主项目。

每个**未跳过**的子模块内执行：

0. **`master` 分支与终止条件**检查（同上）；非 `master` 则 `git checkout master`，失败则 **STOP** 并交给用户。
1. **若工作区有改动**（含未跟踪，按需）：按上文 **「Stash 使用规则」** 使用整轮共用的 `$STASH_TAG`，执行 `git stash push -u -m "$STASH_TAG"`；仅当成功创建 stash 时标记本目录「本轮已 stash」。若 nothing to save，**跳过 stash**，且本目录**禁止**执行 `stash pop`。  
   - **不得**使用与历史 stash 可能重名的短 message（如单独的 `sync-`、用户名、日期）。
2. `git pull origin master`
3. **仅当本目录步骤 1 成功创建了带 `$STASH_TAG` 的 stash**：`git stash pop "stash^{/$STASH_TAG}"`（或按规则核对 message 后的等价显式 ref）；冲突则解决后继续。**禁止**裸 `git stash pop`。
4. 若有需要提交的变更（含冲突解决后的提交）：`git add -A` → `git commit -m "..."`。
5. `git push origin master`（无领先提交时通常 no-op / already up to date）。

**常见边界（与旧流程一致）：**

- **`stash pop` 报 untracked already exists**：`pull` 已带入同路径文件时，工作区可能已正确；仅可核对后 **`git stash drop "stash^{/$STASH_TAG}"`** 丢弃**本轮**条目，勿重复 pop，**勿 drop 其它 message 的 stash**。
- **Windows：`failed to remove ... Directory not empty`**：若本轮 stash 与当前工作区重复，评估后仅 **drop 本条 `$STASH_TAG`**，以当前工作区为准继续；**禁止** drop 历史 stash。

完成后，**凡未在阶段 2 跳过的** consumer 子模块的 `HEAD` 应与阶段 1 未跳过时的远程 `master` 一致（除非刻意保留本地未推送提交——一般不推荐）。已在阶段 2 整 repo 跳过的主项目，其嵌套子模块**本轮不保证**与远程一致。

---

### 阶段 3 — **五个主项目**（主仓库根目录，非子模块内）

对 **`xituan_backend`、`xituan_cms`、`xituan_platform`、`xituan_site`、`xituan_wechat_app`** 各仓库根目录：

**每个主仓库开始前**：在该主仓库根目录**再次**按「主仓库无待同步」全文判定（建议含 `git fetch origin master`；条件与阶段 2 入口一致）。若仍为 **整 repo 跳过**，则**跳过**本阶段该目录的全部步骤，下一主项目。（阶段 2 若曾对该主项目执行子模块 pull 并导致主仓出现 `submodules/xituan_codebase` 等变更，通常此处**不应**再判定为跳过。）

否则：

0. **`master` 分支与终止条件**检查；非 `master` 则 `git checkout master`，失败则 **STOP** 并交给用户。
1. 视需要保持与远程一致：可先 `git pull origin master`（若本地有未提交改动阻碍 pull，必须用 **「Stash 使用规则」** 的同一 `$STASH_TAG`：**本目录** `stash push -m "$STASH_TAG"` → `pull` → **仅** `stash pop "stash^{/$STASH_TAG}"`；**禁止**沿用仓库内旧 stash、**禁止**无参 pop）。
2. 确认子模块指针已更新：`git status` 中 `submodules/xituan_codebase` 等应反映**本轮**阶段 2 已执行的结果（或与阶段 1 未跳过时的远程 `master` SHA 一致）。若该主项目在阶段 2 曾整 repo 跳过，本步仅核对当前指针与业务预期，不要求与阶段 2 对齐。
3. **主仓库变更与子模块指针一并提交并推送**（见上文「主仓库提交范围」）：默认 `git add -A` → `git commit -m "..."` → **`git push origin master`**。  
   - 已干净且与 `origin/master` 同步则跳过 commit/push。  
   - 若用户明确要求仅 bump 子模块，可只 `git add submodules/xituan_codebase` 并单独提交。  
   - `xituan_backend` 若本地已有多个未推送 commit，可在一次 `git push origin master` 中一并推送。

---

### 阶段 4 — `xituan_agent`（最后）

**先做「`master` 分支与终止条件」检查**；通过后：

```bash
cd xituan_agent
git rev-parse --abbrev-ref HEAD   # must be master; else checkout master or STOP
git pull origin master    # optional but recommended if team pushes docs
git add -A
git commit -m "docs(devGuide): ..."   # only if changes
git push origin master
```

---

## 实施检查清单

- [ ] **每个**实际参与同步的仓库路径：已进入 **`master`**，否则已 `checkout master` 成功；失败则已 **终止**并交用户手动处理。
- [ ] **整 repo 跳过**：已按「主仓库无待同步」跳过者，**未**对其执行阶段 1/2/3 中对应段落；未跳过者已完整执行。
- [ ] **Stash**：仅创建/恢复/丢弃带**本轮** `$STASH_TAG` 的条目；未创建 stash 的目录未执行 `pop`；未对历史 `stash@{n}` 做无参 `stash pop`。
- [ ] 阶段 1：若 **未**整 repo 跳过 backend，则 `xituan_backend/submodules/xituan_codebase` 已 **pull → add → commit（若有）→ push `origin master`**；若已跳过则本项 N/A。
- [ ] 阶段 2：对 **未**跳过的 consumer 子模块已 **按 Stash 规则**（`$STASH_TAG`、仅 pop 本轮创建）**stash（若有）→ pull → pop（若有）→ commit（若有）→ push `origin master`**
- [ ] 阶段 3：对 **未**跳过的主项目已 **默认**将主仓工作区变更与子模块指针 **一并** `commit` + **`push origin master`**（除非用户明确「仅 bump 子模块」）
- [ ] 阶段 4：`xituan_agent` 在 **`master`** 上已 **最后** commit + `push origin master`（若有文档变更）
- [ ] 所有 **`git commit` message** 符合「Commit message 约束」：**无** Cursor 相关备注或元信息
- [ ] 全程冲突已由 AI 解决；各**实际执行过**的目标仓库 `git status` 干净且与 `origin/master` 一致（或符合团队分支策略）

## 权限与安全

- 推送需要 **git_write** 与 **network**。
- 子模块远程若使用 `https://$GITHUB_TOKEN@github.com/...`，需本机已配置可用凭据。

## 与本技能相关的用户规则

- 子模块与主仓库分离：共享变更**先**在 `xituan_codebase` 远程落地（阶段 1 起点，**除非** backend 主仓整 repo 跳过则阶段 1 不执行），再在各自**主仓库**提交子模块指针（阶段 3，**未**跳过的主项目）。
- `migrations_stable/` 仅人工晋升；新 SQL 只加在 `xituan_backend/migrations/`（与同步流程独立，但提交 backend 时需遵守）。
