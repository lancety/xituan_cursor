---
name: business-error-line-errors
description: >-
  A/B/AB business error shapes: single-instance code, collection lineErrors, or mixed.
  Mode is chosen by API business domain (not error count). Use for cart checkout validation,
  lineErrors, BusinessError batch errors, mixed order+cart failures, and client i18n display.
---

# Business error lineErrors (A / B / AB)

## Shapes

| Shape | API domain | Response |
|-------|------------|----------|
| **A** | Single instance only | `{ code, itemIds? }` |
| **B** | Multi-item collection | `{ code: umbrella, itemIds, lineErrors: [{ itemId, code }] }` — 1 line still B |
| **AB** | Both domains | `code` (single) + `lineErrors` (collection) when both sides fail |

## Checklist

1. **Classify endpoint**: single / collection / both domains?
2. **Pick validation order**: B-first (cart lines) or A-first (user/order) — document in service.
3. **codebase** (edit `xituan_backend/submodules/xituan_codebase` first):
   - `iBusinessErrorLineItem`, `lineErrors?` on `iBusinessErrorData`
   - `business-error-line.enum.ts` — `COLLECTION_UMBRELLA_CODES`
   - `businessErrorLineUtil` — `classifyErrorShape`, `extractLineErrors`, `toLineErrorMap`
   - `BusinessError.createCollectionLineErrors`, `createMixedError`, `createError` (A)
4. **Service**: collection loop → push `lineErrors`; `catch` preserves `BusinessError.code`.
5. **Controller**: `toApiResponse()` — full `errorData`.
6. **Frontend** (grep consumers per `xituan-codebase-change-scope`):
   - `classifyErrorShape` → `collectionHandler` / `singleHandler` / `mixedHandler`
   - i18n per `eBusinessErrorCode`
7. **Tests**: A, B(len=1), B(len>1), AB payload parsing.
8. **Sync**: `xituan-multirepo-codebase-sync` after codebase commit.

## Backend factories

```typescript
// A
BusinessError.createError(code, itemIds?)

// B — always includes lineErrors even when length === 1
BusinessError.createCollectionLineErrors(umbrellaCode, lineErrors)

// AB — rare; both sides in one payload
BusinessError.createMixedError(singleCode, lineErrors, singleItemIds?)
```

## Frontend handlers (by page context)

| Context | Handler |
|---------|---------|
| Cart validate-checkout | B only — highlight rows + summary |
| Product CRUD | A only — existing modal |
| Checkout submit order | B-first short-circuit; mixedHandler must tolerate AB |

## Umbrella codes

Use `COLLECTION_UMBRELLA_CODES` from `business-error-line.enum.ts` (includes `ORDER_ITEMS_INVALID`, `ORDER_ITEM_INVALID`, `CART_ITEM_NOT_FOUND`, etc.).

## Related

- Rule: `.cursor/rules/business-error-response.mdc`
- Scope: `.cursor/skills/xituan-codebase-change-scope/SKILL.md`
- Sync: `.cursor/skills/xituan-multirepo-codebase-sync/SKILL.md`
