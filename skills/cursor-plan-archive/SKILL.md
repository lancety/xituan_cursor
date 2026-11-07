---
name: cursor-plan-archive
description: >-
  Archive Cursor plan files between ~/.cursor/plans (draft), workspace
  .cursor/plans_planned (deferred/planned-work), and .cursor/plans_completed
  (done). Use at the end of Plan-mode work, when registering planned-work, when
  marking a plan done, or when the user asks to move/archive plans.
---

# Cursor plan 归档（planned / completed）

## 路径

| 桶 | 绝对路径 |
|----|----------|
| 草稿 | `C:/Users/lance/.cursor/plans/` |
| planned | `d:/projects/xituan_module/.cursor/plans_planned/` |
| completed | `d:/projects/xituan_module/.cursor/plans_completed/` |

`.cursor/` 是独立仓 **xituan_cursor**，可直接 git push。

## 何时用

- Cursor **Plan 模式**一轮结束、方案定稿
- 登记 planned-work 或标完成
- 用户说「挪到 planned / completed」

## 收尾询问（先问再动）

用中文简短问清：

1. **留在本机草稿**（还在改）
2. **挪到 `plans_planned`**（定稿、以后做 / 已写台账）
3. **挪到 `plans_completed`**（已落地）
4. **删除**（仅用户明确弃用）

未明确答复前不要 `mv` / `rm`。

## 挪到 plans_planned

```bash
mv "C:/Users/lance/.cursor/plans/<name>.plan.md" \
  "d:/projects/xituan_module/.cursor/plans_planned/"
```

- 提醒是否更新 `xituan_agent/devGuide/planned-work` entry + registry；entry 内 Cursor plan 路径写上表 planned 路径。
- overview 可加 `【planned / high|mid|low】`。

## 挪到 plans_completed

```bash
mv "<from draft or plans_planned>/<name>.plan.md" \
  "d:/projects/xituan_module/.cursor/plans_completed/"
```

- overview 可标 `【completed YYYY-MM-DD】`。
- 对应 registry ID → `done`（若用户要改台账）。

## 查找

```text
C:/Users/lance/.cursor/plans/
d:/projects/xituan_module/.cursor/plans_planned/
d:/projects/xituan_module/.cursor/plans_completed/
```

## 示例询问

> 本轮 plan 已定稿。要挪到工作区 `.cursor/plans_planned`（xituan_cursor 可 git），还是仍留本机草稿？已做完则说一声，我挪到 `plans_completed`。
