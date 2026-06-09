---
name: responsive-table-mobile-card-styling
description: ResponsiveTable mobile card styling—unified margin/border-radius via .mobileContainer > *; do NOT set margin-bottom or border-radius on .mobileCard or custom mobileCardRender cards. Values align with mobile-product-card breakpoints (12/8px default, 8/6px at 768px, 18px at 480px). Use when adding mobileCardRender, custom card CSS, or debugging ResponsiveTable mobile layout.
---

# ResponsiveTable Mobile Card Styling

Reference: `xituan_agent/devGuide/responsive-table-solution.md`. Styles in `xituan_cms/submodules/xituan_codebase/components/ResponsiveTable.module.css`. For pagination logic and behavior, see skill **responsive-table-pagination**.

---

## 1. Single source of truth: .mobileContainer > *

`.mobileContainer > *` provides unified **margin-bottom** and **border-radius** for all list items (default cards and mobileCardRender custom cards). Do **not** override these in child elements.

---

## 2. Do NOT set on .mobileCard or custom cards

- **margin-bottom**: Never set on `.mobileCard` or custom cards (e.g. `mobile-order-card`, `mobile-product-card`). Causes responsive spacing to break.
- **border-radius**: Never set on `.mobileCard` or custom cards. Use `.mobileContainer > *` values.

`.mobileCard` should only have: `transition`, `background`, `border`. No `margin-bottom`, no `border-radius`.

---

## 3. Responsive breakpoints (match mobile-product-card)

| Breakpoint        | margin-bottom | border-radius | .mobileCardBody padding |
|-------------------|---------------|---------------|-------------------------|
| Default           | 12px          | 8px           | 12px                    |
| max-width 768px   | 8px           | 6px           | 8px                     |
| max-width 480px   | 18px          | 6px           | 6px                     |

---

## 4. Custom mobileCardRender

When using `mobileCardRender`, custom cards must omit `margin-bottom` and `border-radius`:

```tsx
// ✓ Correct: no margin-bottom, no border-radius
<Card className="mobile-order-card" style={{ background: '...', border: '...' }}>

// ✗ Wrong: overrides ResponsiveTable
<Card style={{ marginBottom: 12, borderRadius: 8 }}>
```

Page-specific CSS (e.g. order-list.css) must not add margin-bottom or border-radius to list cards; ResponsiveTable handles it.

---

## 5. Quick check

- `.mobileCard`: no margin-bottom, no border-radius.
- Custom cards in mobileCardRender: no margin-bottom, no border-radius.
- Page CSS: no margin-bottom/border-radius overrides for ResponsiveTable list items.
