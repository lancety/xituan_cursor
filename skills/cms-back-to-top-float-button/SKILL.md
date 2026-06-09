---
name: cms-back-to-top-float-button
description: Adds a reusable Back-to-Top button in xituan CMS pages using Ant Design FloatButton.BackTop. Use when implementing or refactoring "返回顶部/BackTop/back to top" UI, aligning the button to a centered content container, or replacing BackTop icons with the standard up-to-top SVG.
---

# CMS Back To Top Float Button

## Use the reusable component

- Prefer using the shared component:
  - `xituan_cms/src/components/common/BackToTopFloatButton.tsx`
- Do **not** re-implement `resize` + `ResizeObserver` alignment logic in each page.

## Quick start (recommended)

1. Create a `ref` for the page content container you want to align against.
2. Render `BackToTopFloatButton` with `alignToRef` and a scoped `className`.

Example:

```tsx
import React, { useRef } from 'react';
import BackToTopFloatButton from '../components/common/BackToTopFloatButton';

export default function Page() {
  const pageContentRef = useRef<HTMLDivElement>(null);
  const enabled = true; // usually depends on list length

  return (
    <div ref={pageContentRef}>
      {/* page content */}

      <BackToTopFloatButton
        enabled={enabled}
        className="partner-supply-backtop"
        alignToRef={pageContentRef}
      />
    </div>
  );
}
```

## Props guidance

- `enabled`: gate rendering; when `false`, component returns `null` (no DOM).
- `alignToRef`: aligns the button to the **right edge** of the referenced container (useful for centered layouts with gutters).
- `right`: fixed right offset when you **do not** need container alignment.
- `visibilityHeight`: when the button appears (default `400`).
- `icon`: optional; default is the standard **up-to-top** SVG using `currentColor`.

## Styling rules (important)

- Keep styles **scoped** by `className` (e.g. `.partner-supply-backtop`) to avoid affecting other FloatButtons.
- Do not hardcode theme colors; rely on CSS variables (e.g. `var(--text-primary)`) and `currentColor`.

### Ant Design default style caveat

Ant Design v5 injects CSS-in-JS styles that commonly include:

- `.ant-float-btn-content { overflow: hidden; }` (can clip large custom SVG)
- `.ant-float-btn-icon { width: 18px; }` (can cause icon offset when using larger icons)

If the SVG is clipped or visually offset, add a **scoped** override in `xituan_cms/src/styles/globals.css` under your `className`, for example:

```css
.partner-supply-backtop .ant-float-btn-content {
  overflow: visible !important;
}

.partner-supply-backtop .ant-float-btn-icon {
  width: 1em !important;
  margin: 0 !important;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

## When to apply this skill

Apply this pattern when:

- A CMS/partner-facing page needs a "返回顶部" button.
- Existing pages use `FloatButton.BackTop` directly and start duplicating alignment logic.
- A custom SVG icon is introduced and you see clipping/offset from Ant's default styles.

