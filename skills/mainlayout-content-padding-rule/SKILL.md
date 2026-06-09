---
name: mainlayout-content-padding-rule
description: MainLayout main content container standard for CMS and Platform pages—1660px wrapper no padding; main section className cms-main-content-wrapper (12px mobile / 24px desktop padding, --bg-secondary); header block and section blocks with cardStyle; title block cms-page-header-card, unified h1 (text-2xl font-bold); list Card (ResponsiveTable wrapper) mobile transparent; ResponsiveTable locale.emptyText for mobile empty state. Use when building or refactoring pages under MainLayout.
---

# MainLayout Main Content Container Standard

Reference: **Orders** (`xituan_cms/src/pages/orders.tsx`), **Payment records** (`payment-records.tsx`), **Revenues** (`revenues/index.tsx`). Full doc: `xituan_agent/devGuide/MainLayout-Main-Content-Container-Standard.md`.

---

## 1. 1660px wrapper: no padding

**The max-width 1660px parent container (the div that wraps `page-container` in MainLayout) must NOT use padding**—including **not** `padding: 24px`. A single padding layer avoids double padding and keeps the layout clean.

Structure:

```
Layout (outer)
  └─ Layout (inner, maxWidth 1900px)
       └─ Content
            └─ div [max-width: 1660px, margin: 0 auto]   ← NO padding (no 24px)
                 └─ div.page-container
                      └─ page content (one wrapper with 24px padding + bg)
```

- In MainLayout do **not** set `padding: 24px` or any padding on the 1660px div.
- Padding is provided by the **main section wrapper** inside the page (see below).

---

## 2. Main section: cms-main-content-wrapper

Under `.page-container`, use `className="cms-main-content-wrapper"`. Global CSS (`globals.css`) provides:

- **Padding**: 12px mobile (max-width 767px); 24px desktop (min-width 768px).
- **Background**: `var(--bg-secondary)`; `width: 100%`; `min-height: 100vh`.
- **Mobile Card body**: Inside this wrapper, `.ant-card .ant-card-body` gets 12px padding on mobile.

Example: `<div className="cms-main-content-wrapper">`. Do not add inline padding; the class handles it.

---

## 3. Card style constant

Use a shared style for header and section cards: `const cardStyle = { background: 'var(--bg-primary)', borderColor: 'var(--border-primary)', borderRadius: 8 };`. Apply `...cardStyle` to every Card used as header or section block.

---

## 4. Main content header block (title block)

- **One dedicated Card** at the top: `className="cms-page-header-card"`, `style={{ marginBottom: 24, ...cardStyle }}`.
- **Title (h1)**: unified style for consistent height and font across pages—`className="text-2xl font-bold text-gray-900 dark:text-white"`, `style={{ margin: 0 }}`.
- **Subtitle** (optional): short description under the title (e.g. `Text` with `color: 'var(--text-secondary)'`, `marginTop: 4`).
- **Right-aligned**: page-level action buttons in the same row as the title.
- **Last child no margin**: The header card body’s last direct child must have no bottom margin. Global CSS in `globals.css`: `.cms-page-header-card.ant-card .ant-card-body > *:last-child { margin-bottom: 0 !important; }`. Do not add `marginBottom` to the last child inside the header Card.

---

## 5. Section blocks: spacing and background

- Each logical section (filters, table, etc.) is a **Card** with `style={{ marginBottom: 24, ...cardStyle }}` (last block may omit marginBottom).
- **Spacing between blocks**: **24px**. Use theme variables only; no hardcoded colors.

### 5.1 List Card (ResponsiveTable wrapper)

For Cards wrapping **ResponsiveTable**, on mobile make the Card transparent: `style={isMobile ? { backgroundColor: 'transparent', border: 'none' } : cardStyle}`, `styles={isMobile ? { body: { backgroundColor: 'transparent', padding: 0, boxShadow: 'none' } } : undefined}`. Always pass `locale={{ emptyText: '...' }}` to ResponsiveTable for mobile empty state. For ResponsiveTable mobile card margin/border-radius rules, see skill **responsive-table-mobile-card-styling**.

---

## 6. Color summary

| Layer           | Background           |
|----------------|----------------------|
| Main section   | `var(--bg-secondary)` |
| Header block   | `var(--bg-primary)`   |
| Section blocks | `var(--bg-primary)`   |

Blocks use `cardStyle` (primary background, `--border-primary`, borderRadius 8). Blocks on primary over secondary keep contrast comfortable in both themes.

---

## 7. When to apply

- Adding or refactoring MainLayout in **xituan_cms** or **xituan_platform**.
- Building new pages with a clear header and multiple sections.
- Reviewing layout when “double padding”, inconsistent main content style, or title block height is reported.

---

## 8. Quick check

- 1660px div: no padding.
- Main section: `className="cms-main-content-wrapper"` (12px mobile / 24px desktop).
- Header Card: `className="cms-page-header-card"`, `marginBottom: 24`, `...cardStyle`; h1 with `text-2xl font-bold text-gray-900 dark:text-white`, `margin: 0`; last child of card body has no margin-bottom.
- Section Cards: `...cardStyle`, 24px gap between blocks.
- List Card (ResponsiveTable): mobile transparent; ResponsiveTable has `locale={{ emptyText: '...' }}`.
