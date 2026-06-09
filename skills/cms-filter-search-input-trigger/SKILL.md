---
name: cms-filter-search-input-trigger
description: Search/filter Input that triggers API queries with IME-safe handling (Pinyin, Japanese, Korean). Use when adding or modifying search keyword inputs in CMS list filter bars, ResponsiveTable filter areas, or any input that debounces API calls.
---

# CMS filter search input: IME-safe query trigger

## When to use this skill

- Adding a search keyword Input to a list page filter bar (products, orders, news, partners, etc.).
- Modifying existing search inputs that trigger API queries on change.
- Debugging issues where: keyword disappears when typing (wrong ref), Pinyin causes unwanted intermediate API calls, or English input behaves oddly with IME active.

## Core pattern

Use **debounced onChange + composition events** so that:
- During IME composition (Pinyin typing), onChange is skipped.
- On compositionend, trigger search with the final value.
- For English (no composition), onChange triggers debounced search as usual.

## Implementation要点

1. **Separate refs** for desktop and mobile Inputs — never share one ref when both exist in DOM (one hidden). Reading from the wrong ref causes keyword to vanish.
2. **isComposingRef**: `onCompositionStart` sets true; `onCompositionEnd` sets false and triggers search.
3. **onChange**: skip when `isComposingRef.current`; otherwise debounced search.
4. **getSearchInputValue()**: read from `searchInputDesktopRef` or `searchInputMobileRef` based on `isMobile`.
5. **Reset**: clear both refs’ values. **Sync from URL**: set the visible input’s value when `filters.keyword` loads.

## Reference

- Full design: `xituan_agent/devGuide/cms-filter-search-input-ime-solution.md`
- Example: `xituan_cms/src/components/products/ProductList.tsx`
