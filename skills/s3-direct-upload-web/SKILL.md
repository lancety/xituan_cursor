---
name: s3-direct-upload-web
description: >-
  Browser direct upload for CMS and Site: fetch PUT of a File/Blob to the opaque
  putUrl, with Content-Type match, checksum-safe presign, and short retries.
  Use when a web page uploads bytes with presign/putUrl (expense batch OCR,
  future catalog images). Read s3-direct-upload first for the shared contract.
---

# S3 direct upload — web

Read **s3-direct-upload** first. This skill is only the browser PUT.

## Shipped reference

`xituan_cms/src/components/expenses/ExpenseBatchOcrPanel.tsx` → `putFileToPresignedUrl`.

That path PUTs to a presigned `amazonaws.com` URL from `POST /admin/expenses/batch/presign`, then notifies the backend. Catalog/OpenIM keep this PUT shape. The server chooses `putUrl` (see [s3-direct-upload/reference.md](../s3-direct-upload/reference.md)). Do not branch the component on hostname.

## PUT

```ts
await fetch(putUrl, {
  method: 'PUT',
  headers: { 'Content-Type': contentType },
  body: file,
});
```

- `body` is the `File` / `Blob`. Not `FormData`. Not a multipart body.
- `contentType` is the same string sent to presign (`file.type`). Empty type on expense falls back to `image/jpeg` only because that API rejects non-images.
- Leave `credentials` at the default (`omit`). Do not send cookies to S3 or the relay.
- Do not follow a redirect onto another host; the signature will not match.
- Treat non-2xx as failure. Do not read the object URL from the PUT response; wait for `complete` (expense uses notify + `s3Key` — do not copy that into catalog/OpenIM).

## Retries

Expense retries the PUT up to 3 times with linear backoff (`400 * attempt` ms) for transient `net::ERR_CONNECTION_RESET`. Reuse that for idempotent PUTs to the same key. Do not retry `complete` until the PUT returned 2xx. A checksum-poisoned URL will fail every attempt — fix the signer (see shared skill), do not add more retries.

## CORS

If `putUrl` is cross-origin (`amazonaws.com` or `content.*`), that host must allow `PUT` and the `Content-Type` header from the page origin. A missing CORS rule looks like a network failure in `fetch`, not an HTTP status.

## Page duties

Pick file → call the codebase orchestrator (or, until it exists, the same three calls) → store only the canonical path on the form. Submit has no file field.

## Site vs CMS transport (pitfalls)

Shipped: `xituan_site/src/utils/direct-upload-web.transport.ts`,
`xituan_cms/src/utils/direct-upload-web.transport.ts`.

### API base — do not double `/api`（必查）

Both Site and CMS `NEXT_PUBLIC_API_URL` typically already end with `/api`.

| App | Join path (copy sibling clients) |
|-----|----------------------------------|
| Site | `{API_BASE}/direct-upload/presign` |
| CMS | `{API_BASE}/admin/direct-upload/presign` (same prefix style as `product.api` → `/admin/products`) |

**Forbidden:** `{API_BASE}/api/...` when base already has `/api`  
→ `…/api/api/admin/direct-upload/presign` → **404**.

**Before commit:** open DevTools Network (or read env + path) and confirm the
full URL has a **single** `/api`. Do not invent CMS as “host-only + `/api/admin`”
unless that app’s other `fetch` calls literally do the same.
### Site Authorization

Site tokens live in `authStoreManager` (`localStorage` key `xituan_auth` JSON),
**not** bare `localStorage.token` / `accessToken`.

Use `authStoreManager.getToken()` (same as `cart.service` / `auth.service`).
Missing Bearer still hits customer authenticate → 401 `TOKEN_REQUIRED`.

### Parent append vs child file input

If the parent `setNoteImageFiles(prev => [...prev, ...files])`, the child file
`onChange` must pass **only newly selected files**, not
`[...existing, ...new]`. Passing the full list double-appends (preset note image
duplicated when user adds another photo).

### no-wait thumbs (Site cart notes — standard)

When upload uses `waitNormalize: false` and the next page shows CDN thumbs:

1. Prefer `_w*` / `getImageUrl(..., s64)`.
2. **`img.onError` → `getOriginalUrl(path)`** (promote 后原图已在 media).
3. Preview/lightbox already uses original — no change.

Do not sync-wait Lambda on add→cart. Full rule: parent skill **no-wait display standard**.

Reference: `siteImageUtil.getCartNoteThumbUrl` + `CartItemCard` / `CartItemRow`.
