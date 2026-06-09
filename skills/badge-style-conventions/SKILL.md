---
name: badge-style-conventions
description: >-
  badge 状态样式 (soft .badge-state) vs badge 按钮样式 (filled .badge-btn). Canonical CSS vars
  --badge-state-* / --badge-btn-* in xituan_cms/src/styles/badge-style.css; shared enums/utils in
  xituan_codebase (badge-style.enum.ts, badge-style-css-vars.enum.ts, badge-style.util.ts).
  Use when the user says badge 状态样式, badge 按钮样式, 状态 badge, 实心 badge, pill, chip,
  or unifying CMS status/publish badges. Do not use Ant Design Tag for badge buttons.
---

# Badge 样式约定（xituan 共享实现）

## 触发词（必须遵守）

| 用户说法 | 模式 | 类名 | CSS 变量前缀 |
|---------|------|------|----------------|
| **badge 状态样式** / 状态 badge / soft pill / chip 状态 | Soft，低对比只读 | `.badge-state` + `.badge-state--{tone}` | `--badge-state-{tone}-{fg\|bg\|border}` |
| **badge 按钮样式** / 实心 badge / 发布状态样式 | Filled，强调/可点击 | `.badge-btn` + `.badge-btn--{color}`（`span`/`button`，**不用** Ant `Tag`） | `--badge-btn-{color}-{fg\|bg\|border}` |

> 只说「badge」且未指定时：**先问**状态还是按钮；表格只读状态列 → **badge 状态样式**；发布状态/详情 CTA → **badge 按钮样式**。

---

## 共享实现（codebase + CMS，优先查阅）

| 用途 | 路径 |
|------|------|
| **CSS 变量 + 工具类** | `xituan_cms/src/styles/badge-style.css`（`_app.tsx` 已全局引入） |
| **Tone / color 枚举** | `xituan_codebase/constants/badge-style.enum.ts` → `epBadgeStateTone`, `epBadgeBtnColor` |
| **CSS var 名清单（AI 依赖）** | `xituan_codebase/constants/badge-style-css-vars.enum.ts` → `BADGE_STATE_CSS_VARS`, `BADGE_BTN_CSS_VARS` |
| **className 助手** | `xituan_codebase/utils/badge-style.util.ts` → `badgeStyleUtil.stateClass`, `btnClass`, `btnLinkClass` |

### 状态 tone（`epBadgeStateTone`）

`positive` | `caution` | `danger` | `neutral` | `muted`

订单/支付等旧名映射：`success→positive`, `warning→caution`, `error→danger`（`badgeStyleUtil.stateClassFromSemanticAlias`）

### 按钮 color（`epBadgeBtnColor`）

`blue` | `green` | `orange` | `red` | `default` | `cyan` | `purple`

---

## 代码用法

### badge 状态样式

```tsx
import { epBadgeStateTone } from '.../badge-style.enum';
import { badgeStyleUtil } from '.../badge-style.util';

<span className={badgeStyleUtil.stateClass(epBadgeStateTone.CAUTION)}>待处理</span>
```

自定义 CSS 引用变量（勿写死色值）：

```css
.my-row {
  color: var(--badge-state-caution-fg);
  background: var(--badge-state-caution-bg);
  border-color: var(--badge-state-caution-border);
}
```

或 TypeScript：`BADGE_STATE_CSS_VARS[epBadgeStateTone.CAUTION].bg` → `'--badge-state-caution-bg'`

### badge 按钮样式

```tsx
import { epBadgeBtnColor } from '.../badge-style.enum';
import { badgeStyleUtil } from '.../badge-style.util';

<span className={badgeStyleUtil.btnClass(epBadgeBtnColor.BLUE)}>已发布</span>
```

**badge 按钮内的 `<a>`**（防全局链接色覆盖）：

```tsx
<span className={badgeStyleUtil.btnClass(epBadgeBtnColor.BLUE)}>
  <a href="..." className={badgeStyleUtil.btnLinkClass()}>详情</a>
</span>
```

外层须为 `.badge-btn` + `.badge-btn--{color}`（`btnClass()`）；内链用 `btnLinkClass()`（`badge-btn__link`）。

**避免被全局 `<a>` 盖色**（不依赖 `ant-table` / `ant-tag`）：
- `globals.css` 表格链接规则已排除 `:not(.badge-btn):not(.badge-btn__link)`
- `badge-style.css` 用 `.badge-btn.badge-btn--{color}` 提高优先级
- 单元素链接：`<a className={\`${badgeStyleUtil.btnClass(BLUE)} ${badgeStyleUtil.btnLinkClass()}\`} />`

---

## 明暗主题

变量定义在 `badge-style.css` 的 `[data-theme='light']` / `[data-theme='dark']`（及 `body[data-theme=…]`）。

- **状态**：背景比旧 `cms-semantic-pill` **更淡**（亮色更浅纯色底，暗色 rgba ≈ 0.05–0.06）
- **按钮**：bg/border 保持饱和；**fg 更亮**（暗色用 `#f0f9ff` 等浅字）

---

## 遗留兼容

| 旧名 | 新名 |
|------|------|
| `.cms-semantic-pill--success` | `.badge-state--positive`（同变量） |
| `.status-tag-outlined`（已废弃） | `.badge-btn--{color}` |
| `--cms-semantic-pill-*` | 别名 → `--badge-state-*`（`globals.css`） |
| `cms-tag-solid-btn` | 布局在 globals；颜色别名 `--badge-btn-blue-*` |

新功能**禁止**新增 `--cms-semantic-pill-*` 或硬编码 `#dbeafe`；用 `BADGE_*_CSS_VARS` 或 `badgeStyleUtil`。

---

## Agent 清单

1. 识别用户要 **badge 状态样式** 还是 **badge 按钮样式**（上表）。
2. 打开 `badge-style.css` + `badge-style-css-vars.enum.ts` 确认变量名。
3. TSX 用 `badgeStyleUtil` / `epBadgeStateTone` / `epBadgeBtnColor`；CSS 用 `var(--badge-…)`。
4. 明暗主题两套值已在 `badge-style.css` 维护，组件内勿写死 `#fff` / `#1e3a8a`。
5. 改 `xituan_codebase` 导出时，按 `xituan-codebase-change-scope` 更新实际 import 的 consumer（通常 CMS）。

---

## 活动订单卡片（示例）

- 订单/支付状态：`activityOrderDetailModalUtil.stateBadgeClass(tone)` → `.badge-state--*`
- 详情链接：`span` + `btnClass(BLUE)` + 内链 `btnLinkClass()`
- 发布状态列：`OfferLifecycleTag` / `PreorderLifecycleTag` → `span` + `btnClass(color)`
