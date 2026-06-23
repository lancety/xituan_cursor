---
name: responsive-table
description: Use the shared ResponsiveTable from xituan_codebase for desktop Table plus mobile Cards, including fixed columns, mobileCardRender, infinite loading, pagination URL/query sync, sortConfig, and current card styling rules.
---

# ResponsiveTable

Use the shared `ResponsiveTable` for list pages that must work as an Ant Design Table on desktop and mobile cards below the configured breakpoint.

## Import And Scope

- Source: `submodules/xituan_codebase/components/ResponsiveTable.tsx`.
- CMS import often uses a relative submodule path.
- Platform import often uses the alias: `import ResponsiveTable from 'xituan_codebase/components/ResponsiveTable';`.

## Current Capabilities

- Desktop renders Ant Design `Table`; normal `Table` props pass through.
- Mobile renders cards when `!screens[mobileBreakpoint]`; default breakpoint is `md`.
- Column extensions:
  - `mobileShow`;
  - `mobileRender`;
  - `mobileLabel`;
  - `mobilePriority`;
  - `enableSort`.
- Custom mobile layout: `mobileCardRender(record, index)`.
- Scroll/infinite mode: pass `onLoadMore` and `hasMore`; this disables pagination and uses the sentinel flow.
- Pagination mode:
  - default `pageSizeOptions` is `[10, 20, 50]` if caller does not pass one;
  - `paginationScrollCompensation` is enabled by default when pagination `onChange` returns a Promise;
  - `paginationUrlQuery` can sync client-side pagination to the URL, or be forced with `true` / options.
- Sorting mode: pass `sortConfig` with current `sortBy`, `sortOrder`, and `onChange`.

## Desktop Fixed Columns

- Fixed left columns: `fixed: 'left'` and explicit `width`.
- Fixed right action columns: `fixed: 'right'` and explicit `width`.
- Always pass `scroll={{ x: number }}` or `scroll={{ x: 'max-content' }}` when fixed columns are used.
- Keep action buttons on one line (`Space wrap={false}` where appropriate).

## Mobile Cards

- Use `mobileShow: false` for noisy columns.
- Use `mobilePriority` so important fields appear first.
- Use `mobileRender` when the desktop `render` output is too complex for cards.
- Use `mobileCardRender` for fully custom cards such as Orders or Offers.
- Always pass `locale={{ emptyText: '...' }}` so mobile empty state is clear.

## Mobile Styling Rules

- `ResponsiveTable.module.css` provides list item margin and border radius via `.mobileContainer > *`.
- Do not set `margin-bottom` or `border-radius` on default `.mobileCard` or custom `mobileCardRender` cards.
- Custom cards should use theme variables and, when useful, class `responsive-table-mobile-card`.
- For a parent `Card` wrapping `ResponsiveTable`, use the MainLayout list-card pattern: desktop `cardStyle`, mobile transparent card body with padding 0.

## Pagination Rules

- For server pagination, keep one handler: `onChange: (page, size) => handlePageChange(page, size)`.
- Do not also pass `onShowSizeChange`; Ant Design fires both and can duplicate requests.
- Use `usePaginationPageSizeStorage(storageKey, defaultPageSize)` when page size should persist.
- Use `calcPageAfterSizeChange(currentPage, oldPageSize, newPageSize)` to keep the first visible item stable.
- Return a Promise from the page-change handler so scroll compensation runs after data refresh.

## References

- `xituan_agent/devGuide/responsive-table-solution.md`
- `.cursor/skills/responsive-table-pagination/SKILL.md`
- `.cursor/skills/responsive-table-mobile-card-styling/SKILL.md`
- `xituan_codebase/components/ResponsiveTable.tsx`
