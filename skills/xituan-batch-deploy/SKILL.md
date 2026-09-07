---
name: xituan-batch-deploy
description: >-
  Batch-deploys xituan multi-repos: tsc_lint (cms/site when in scope), confirm master,
  multirepo sync to origin/master, then checkout production → align with master →
  push origin production → checkout master. Keywords: 批量部署, batch deploy. If the user
  names repos, only those; if none named, all deployable main repos (skip repos with no
  production branch for the production push step). Uses xituan-multirepo-codebase-sync
  rules for stash/master/commit messages.
---

# Xituan 批量部署（batch deploy）

触发词：**批量部署** / **batch deploy**。

`xituan_module` 根不是单一 Git 仓。本流程在**各独立主仓库**上：先把 `master` 同步到 `origin`，再把 `production` 对齐并推送（触发生产部署），最后切回 `master`。

## 仓库范围

| 用户说法 | 范围 |
|----------|------|
| 未提具体 repo | **全部**可部署主仓：`xituan_backend`、`xituan_cms`、`xituan_platform`、`xituan_site`、`xituan_wechat_app` |
| 点名 repo（如「只做 cms 和 site」） | **仅**点名的主仓 |

**非主仓 / 无 production：**

- `xituan_agent`、`.cursor`（`xituan_cursor`）：**不参与** production 推送；仅当用户明确要求或全量 multirepo sync 时走 `xituan-multirepo-codebase-sync` 的 agent/cursor 阶段。
- 某主仓**没有**本地/远程 `production` 分支：该仓**跳过** production 步骤，在报告中写明「无 production，已忽略」。

别名：`cms`→`xituan_cms`，`site`→`xituan_site`，`backend`→`xituan_backend`，`platform`→`xituan_platform`，`wechat`→`xituan_wechat_app`。

---

## 固定阶段（必须按序）

### 0 — 前置 lint（有 TS 前端仓时）

对范围内的 **`xituan_cms`** / **`xituan_site`**（若在范围内）：

```bash
cd <repo>
npm run tsc_lint
```

- **错误（exit ≠ 0）**：先修复再继续；修完复跑直到通过。
- **仅 Warning、exit 0**：视为通过，不强制清 warning。
- 范围内无 cms/site：跳过本阶段。

### 1 — 确认 `master` + multirepo sync（仅范围内仓）

**先读并遵循** `.cursor/skills/xituan-multirepo-codebase-sync/SKILL.md`：

- 每个 Git 路径必须在 **`master`**（失败则 **STOP**）。
- Stash 仅用本轮 `$STASH_TAG`；commit message **禁止** Cursor 相关字样。
- **子集**：用户点名时，只对点名主仓及其 `submodules/xituan_codebase` 执行阶段 2/3；**不要**动未点名主仓。
  - 若子集**不含** `xituan_backend`：跳过 sync 技能的「阶段 1 backend codebase」；consumer 子模块仍各自 `pull origin master`（与远程 codebase 对齐），再提交主仓指针+业务变更。
  - 若子集含 backend：仍按 sync 技能阶段 1→2→3 对该子集执行。
- 主仓默认：`git add -A` → commit（若有）→ **`git push origin master`**。
- 干净且已与 `origin/master` 一致：可跳过该仓 commit/push。

### 2 — production 推送（逐仓）

对范围内**每个**主仓，在阶段 1 成功且当前在 `master`、工作区干净后：

```bash
cd <repo>
git fetch origin master production
# 若 origin/production 或本地 production 不存在 → 跳过本仓 production，报告后下一仓

git checkout production
git pull origin production          # 对齐远程 production
git merge master                    # 期望 fast-forward；冲突则解决后再继续
git push origin production
git checkout master
```

**规则：**

- **禁止**对无 `production` 的仓强行创建/推送 production（除非用户明确要求建分支）。
- **禁止** `push --force` 到 `production`（除非用户明确要求）。
- merge 冲突：在该仓解决后继续；无法安全解决则 **STOP** 并交用户，**不要**留在 production 不管。
- 每个仓结束后必须回到 **`master`**（`git rev-parse --abbrev-ref HEAD` = `master`）。

### 3 — 收尾报告

向用户列出每个范围内仓：

| 仓 | master push | production push | 最终分支 | 备注 |
|----|-------------|-----------------|----------|------|
| … | ok / skipped | ok / skipped(无 production) / failed | master | … |

---

## 检查清单

- [ ] 范围已解析（全量或用户点名）
- [ ] cms/site（若在范围内）`tsc_lint` 已通过
- [ ] 范围内仓已在 master 完成 sync + `push origin master`（或跳过）
- [ ] 有 production 的仓已 `merge master` + `push origin production` + 回到 master
- [ ] 无 production 的仓已跳过并说明
- [ ] 未对未点名仓做 git 写操作

## 权限

- 需要 **git_write** + **network**（pull/push）。

## 与其它技能

- Git 细节（stash、整 repo 跳过、commit 约束）：**xituan-multirepo-codebase-sync**
- 共享代码改动影响面：**xituan-codebase-change-scope**（本流程默认不探索业务代码）
