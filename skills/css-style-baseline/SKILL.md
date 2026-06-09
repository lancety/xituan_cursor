---
name: css-style-baseline
description: Enforces xituan_site CSS baseline rules (minimum font size 14px, spacing, layout containers, and theme-variable usage). Use when editing `globals.css`, `theme.css`, Ant Design overrides, or any UI text styling to prevent tiny fonts and inconsistent base styles.
---

# CSS Style Baseline (xituan_site)

## When to apply

Apply this skill whenever you:

- Change typography (font-size / line-height) or add new text UI
- Add or modify CSS variables in `src/styles/theme.css`
- Add or modify styles in `src/styles/globals.css`
- Override Ant Design component styles (including `.ant-*`)
- Adjust layout containers, header/main spacing, or responsive breakpoints

## Non-negotiable baseline rules

### 1) Minimum font size

- **Minimum font size is 14px** across the entire site.
- No text (including badges, helper text, placeholders, chips, menus, meta text) may be smaller than 14px.
- Do not introduce tokens or CSS variables below 14px.

**Exception:** Font size below 14px is allowed only when:
  - There is an explicit **business requirement** for smaller text, or
  - The **AI user (developer) actively requests** smaller font.

**Implementation requirements**

- Keep design tokens consistent:
  - `--font-size-xs` **must be >= 14px**
  - `--font-size-sm` **must be >= 14px**
- If a UI needs to look visually smaller, use spacing/weight/color, not < 14px.

### 2) Theme color handling (no hardcoded colors)

- Use the existing CSS variables (e.g. `var(--text-primary)`, `var(--bg-base)`, `var(--border-default)`).
- Do not hardcode colors except brand colors when explicitly required.
- Dark/light theme must work via `body[data-theme='light']` and `body[data-theme='dark']`.

### 3) Container width and page padding

- All site layouts must respect the shared max-width container:
  - Use `--site-max-width` and `.site-container`.
- Avoid duplicate padding layers:
  - Prefer a single padding source (typically `.site-container`) unless a block explicitly needs its own padding.

### 4) Header and body must never overlap

- If header height is not fixed, do **not** rely on `calc(100vh - fixedHeaderHeight)`.
- Header content must either:
  - stay in normal flow (preferred), or
  - if sticky/fixed is used later, reserve spacing using a predictable header height variable.

### 5) Responsive behavior should follow classic patterns

- Desktop: topbar utilities + mainbar search (Taobao-like) for consumer pages.
- Mobile: reduce topbar links; show a single “menu” trigger; keep search prominent.
- Do not invent unique layouts per page; reuse template patterns.

## Implementation checklist (before finishing a CSS change)

- [ ] Search for any `font-size` below 14px in changed files.
- [ ] Confirm tokens `--font-size-xs` / `--font-size-sm` are >= 14px.
- [ ] Ensure new colors use theme variables (no `#fff`, `#000`, etc.).
- [ ] Confirm layout stays within `--site-max-width`.
- [ ] Confirm header + body do not overlap at all breakpoints.

