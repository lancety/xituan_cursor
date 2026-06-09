---
name: responsive-table-pagination
description: ResponsiveTable pagination logic—pageSizeOptions default [10,20,50], usePaginationPageSizeStorage for caching, single onChange handler (no onShowSizeChange), calcPageAfterSizeChange, keep-pagination-position scroll compensation. Use when adding or configuring pagination for ResponsiveTable in CMS pages.
---

# ResponsiveTable Pagination

Reference: `ProductList` (`xituan_cms/src/components/products/ProductList.tsx`), `xituan_cms/submodules/xituan_codebase/hooks/usePaginationPageSizeStorage.ts`, `ResponsiveTable.tsx`.

---

## 1. Default pageSizeOptions

ResponsiveTable injects `pageSizeOptions: [10, 20, 50]` when pagination is an object and the caller does not pass `pageSizeOptions`. Pages that need larger options (e.g. partner-supply-list) pass `pageSizeOptions: [10, 20, 50, 100]` explicitly.

---

## 2. Page size caching (localStorage)

Use `usePaginationPageSizeStorage(storageKey, defaultPageSize)` to persist per-page page size preference:

```ts
const [pageSize, setPageSize] = usePaginationPageSizeStorage('products', 20);
```

- Unique `storageKey` per table: `'products'`, `'orders'`, `'payment-records'`, etc.
- `setPageSize(size)` updates state and writes to `localStorage['cms-table-page-size:{storageKey}']`
- On mount, initial `pageSize` is read from localStorage or falls back to `defaultPageSize`

---

## 3. Single onChange handler (no onShowSizeChange)

Ant Design Pagination fires **both** `onShowSizeChange` and `onChange` when page size changes. Passing both causes duplicate API requests and wrong display. Use **only** `onChange(page, pageSize)`:

```tsx
// ✓ Correct: single handler, both page and pageSize
onChange: (page: number, size?: number) => handlePageChange(page, size)

// ✗ Wrong: causes duplicate requests (onShowSizeChange + onChange both fire)
onChange: (page) => handlePageChange(page)
onShowSizeChange: (_, size) => handlePageChange(1, size)
```

---

## 4. calcPageAfterSizeChange

When user changes page size, compute target page so the first visible item stays in view:

```ts
import { calcPageAfterSizeChange } from '.../hooks/usePaginationPageSizeStorage';

// In handlePageChange:
if (newPageSize != null && newPageSize !== pageSize) {
  setPageSize(newPageSize);
  targetPage = calcPageAfterSizeChange(currentPage, pageSize, newPageSize);
}
fetchData(targetPage, limit);
```

---

## 5. Keep pagination position after change (built-in)

After content updates (fewer/more rows), the pagination may move and cause visible jump. ResponsiveTable provides built-in scroll compensation.

**Default**: `paginationScrollCompensation` is true. Parent's onChange must return a Promise that resolves when fetch completes. Pass `paginationScrollCompensation={false}` to disable.

```tsx
<ResponsiveTable
  pagination={{
    onChange: (page, size) => handlePageChange(page, size)  // handlePageChange returns Promise
  }}
/>

// handlePageChange: async (page, size) => { ... await fetchData(...); }
```

Util: `paginationScrollCompensation.util.ts` exports `paginationScrollCompensationUtil.compensate(containerEl, targetTop)` for custom use.

---

## 6. Request versioning (avoid race)

When multiple requests can be in flight (e.g. filters effect + pagination change), use a version ref so only the latest response is applied:

```ts
const fetchVersionRef = useRef(0);
// In fetch: const version = ++fetchVersionRef.current;
// Before applying response: if (version !== fetchVersionRef.current) return;
```

---

## 7. Configuration checklist

| Behavior | How to configure |
|----------|------------------|
| Cache page size | `usePaginationPageSizeStorage('unique-key', 20)` |
| Custom page size options | Pass `pageSizeOptions: [10, 20, 50, 100]` in pagination |
| Single onChange | Use `onChange: (page, size) => handlePageChange(page, size)` only |
| Keep scroll position | Default on; onChange returns Promise. Disable with `paginationScrollCompensation={false}` |
| Backend cap sync | If response.pageSize !== requested, call setPageSize(response.pageSize) |

---

## 8. Quick reference: ProductList / Orders pattern

- `usePaginationPageSizeStorage('products', 20)` for page size caching
- `handleProductPageChange(page, newPageSize?)` is async and returns fetch Promise
- ResponsiveTable (default paginationScrollCompensation) handles scroll after Promise resolves
- `onChange: (page, size) => handlePageChange(page, size)` — no extra refs or callbacks needed
