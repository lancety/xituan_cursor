---
name: async-toggle-switch
description: AsyncToggleSwitch component for table/list item toggles that need async API calls—manages loading and optimistic state internally to avoid full table re-render. Use when adding Switch buttons in list/table rows, toggles for publish/supply/visibility that call APIs, or when the user mentions Switch causing table re-render issues.
---

# AsyncToggleSwitch

## When to use

- **Table/list row toggle** that triggers an async API call (e.g. 供货、已公开/未公开、发布状态)
- **Avoid full table re-render** when user toggles—only the switch re-renders
- User reports "Switch 导致整个 table 重渲染" or similar

## When NOT to use

- **Form fields** (Ant Design Form `Switch`)—use plain `<Switch />` with `valuePropName="checked"`
- **No async call**—use plain Ant Design `Switch` if toggle is purely local/sync

## Component location

```
xituan_codebase/components/AsyncToggleSwitch.tsx
```

Import from submodule, e.g. in xituan_cms:

```tsx
import AsyncToggleSwitch from '../../../submodules/xituan_codebase/components/AsyncToggleSwitch';
```

## Props

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `checked` | `boolean` | yes | Current value from record |
| `onChange` | `(checked: boolean) => Promise<void>` | yes | Async handler—call API here, do NOT update parent state |
| `onError` | `(message: string) => void` | no | Show error, e.g. `(msg) => message.error(msg)` |
| `checkedChildren` | `ReactNode` | no | Label when checked (e.g. "已公开", "供货") |
| `unCheckedChildren` | `ReactNode` | no | Label when unchecked (e.g. "未公开", "—") |
| `size` | `'small' \| 'default'` | no | Default `'small'` |

## Usage example

```tsx
<AsyncToggleSwitch
  checked={!!record.isReleased}
  onChange={async (checked) => {
    if (checked) await api.publish(id);
    else await api.unpublish(id);
  }}
  onError={(msg) => message.error(msg)}
  checkedChildren="已公开"
  unCheckedChildren="未公开"
/>
```

**Critical**: `onChange` must return `Promise<void>`. If the API returns data, wrap:

```tsx
onChange={async (checked) => { await productApi.updatePartnerSupplyItem(id, checked); }}
```

## Current usages

- ProductList: 供货 toggle
- PreorderListItem / OfferListItem: 已公开/未公开
- NewsList: 已发布/草稿
