---
name: badge-style-conventions
description: >-
  badge 状态样式 (soft .badge-state) vs badge 按钮样式 (filled .badge-btn). Visual contract
  lives in xituan_agent/devDesign/badge.md. Implementation: CMS/Site badge-style.css +
  xituan_codebase badge-style enums/utils. Use when the user says badge 状态样式, badge 按钮样式,
  状态 badge, 实心 badge, pill, chip, or unifying CMS status/publish badges.
  Do not use Ant Design Tag for badge buttons.
---

# Badge 实现入口（瘦 skill）

## 视觉权威（先读）

**[`xituan_agent/devDesign/badge.md`](../../xituan_agent/devDesign/badge.md)** — 状态 vs 按钮、tone/color、亮暗、选用规则。  
本 skill **不重复**视觉长文；只保留改代码路径与 checklist。

## 触发词 → 模式

| 用户说法 | 模式 |
|---------|------|
| badge 状态样式 / soft pill / 状态 badge | `.badge-state` + `--badge-state-*` |
| badge 按钮样式 / 实心 / 发布状态样式 | `.badge-btn` + `--badge-btn-*`（**不用** Ant `Tag`） |

只说「badge」时：先问状态还是按钮；表格只读 → 状态；可点/发布 → 按钮。

## 共享实现路径

| 用途 | 路径 |
|------|------|
| CSS 变量 + 工具类 | `xituan_cms/src/styles/badge-style.css`（Site 有对齐的 `badge-style.css`） |
| Tone / color 枚举 | `xituan_codebase/constants/badge-style.enum.ts` |
| CSS var 名清单 | `xituan_codebase/constants/badge-style-css-vars.enum.ts` |
| className 助手 | `xituan_codebase/utils/badge-style.util.ts` → `stateClass` / `btnClass` / `btnLinkClass` |

## 代码用法（短）

```tsx
<span className={badgeStyleUtil.stateClass(epBadgeStateTone.CAUTION)}>待处理</span>
<span className={badgeStyleUtil.btnClass(epBadgeBtnColor.BLUE)}>已发布</span>
<span className={badgeStyleUtil.btnClass(epBadgeBtnColor.BLUE)}>
  <a href="..." className={badgeStyleUtil.btnLinkClass()}>详情</a>
</span>
```

CSS：`var(--badge-state-caution-bg)` 等；或 `BADGE_STATE_CSS_VARS[...]`。  
链接防盖色：外层 `btnClass`，内链 `btnLinkClass`；详见 CMS `globals.css` 排除规则。

## Agent 清单

1. 读 [`devDesign/badge.md`](../../xituan_agent/devDesign/badge.md) 确认模式与 tone/color。
2. 打开对应 `badge-style.css` + `badge-style-css-vars.enum.ts`。
3. TSX 用 `badgeStyleUtil` / 枚举；勿写死色值。
4. 改 codebase 导出时跟 `xituan-codebase-change-scope`。
5. 订单卡业务规则 → skill **`cms-order-card`**（视觉仍服从 `badge.md`）。
