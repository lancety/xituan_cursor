---
name: image-normalize-variant-policy
description: >-
  IMAGE_NORMALIZE per-kind variant policy (allowed `_w*` sizes + webp/png), display URL
  APIs (getContentUrlImageForKind, siteImageProgressiveUtil), enqueue/backfill order, and
  media vs SIH domain split. Use when building or reviewing image display URLs, progressive
  loading, thumbnail sizes, expense receipts, product/news/offer/preorder/avatar/openim images,
  IMAGE_NORMALIZE_VARIANT_POLICY, getContentUrlImage, wechatImageProgressiveUtil, or
  changing which sizes a bind kind may use.
---

# Image normalize variant policy & display URLs

## When to use

- Any **frontend/backend code that builds image URLs** for display (Site / CMS / WeChat / Platform / PDF).
- Changing **allowed sizes or format** for a bind kind.
- Enqueue / backfill / prune of `_w*` variants.
- Reviewing whether call sites follow policy (compile-time + runtime checks).

Upload FormData retention → use **cms-form-image-upload**. CDN hostnames (`media` / `imagesih`) → **media-cdn-sih-domain-split** (`xituan_agent/devGuide/media-cdn-sih-domain-split.md`).

## Source of truth (codebase)

| Piece | Path |
|-------|------|
| Bind kinds | `xituan_codebase/constants/image-normalize-bind-kind.enum.ts` → `epImageNormalizeBindKind` |
| Policy table | `xituan_codebase/utils/image-normalize-variant-policy.util.ts` → `IMAGE_NORMALIZE_VARIANT_POLICY`, `imageNormalizeVariantPolicyUtil` |
| Edge type | `tImageNormalizeVariantEdgeForKind<K>` |
| Progressive-capable kinds | `tImageNormalizeProgressiveKind` (must include 64+128+512) |
| Kind-aware URL | `contentUtil.getContentUrlImageForKind` |
| Unchecked URL | `contentUtil.getContentUrlImage` (**no** policy assert) |
| Canonical / lightbox | `contentUtil.getContentUrl` / Site `getPreviewImageUrl` / `getOriginalUrl` |
| Web progressive | `siteImageProgressiveUtil.resolveUrls` → ForKind |
| Site wrapper | `xituan_site/.../site-image.util.ts` → `getProgressiveUrls(path, approx, kind)` |
| CMS wrapper | `xituan_cms/.../cms-image.util.ts` → same progressive pattern |
| WeChat progressive | `xituan_wechat_app/utils/wechat-image-progressive.wechat.util.ts` (**must** stay aligned with web; prefer ForKind + kind) |
| Type smoke | `image-normalize-variant-policy.type-test.ts` |

Do **not** invent parallel size tables in app code. Enum edges: `enSiteImageSize` (prefer props, never hardcode `64` as magic string where enum exists).

## Hard rules

1. **Display sized URLs must use `getContentUrlImageForKind` (or progressive util that calls it)** with the correct `epImageNormalizeBindKind`.
2. **Width/height must be edges allowed for that kind** — compile-time via `tImageNormalizeVariantEdgeForKind<K>`; runtime via `assertEdgeAllowed` (throws if illegal).
3. **Never use `getContentUrlImage` for kind-restricted assets** (especially `expense_receipt` only `256`). Prefer eliminating bare `getContentUrlImage` for new display code.
4. **Lightbox / full preview** = canonical object (`getContentUrl` / preview helpers), **no** `?width=` / SIH query.
5. **Frontend must not call SIH** for normal UI. Domain key: `media` / `wechatMedia`. SIH hosts (`SIH` / `wechatSIH` / `imagesih-*`) are **backend-only**.
6. **Change size policy order** (non-negotiable):
   1. Edit `IMAGE_NORMALIZE_VARIANT_POLICY` (+ sync Lambda `VARIANT_POLICY` if applicable)
   2. Pre-gen / backfill that kind (`jobs:backfill-normalize` or enqueue path)
   3. Prune orphans if removing edges (`--prune-orphans`)
   4. Then wire UI to the new edge
7. Progressive kinds only: use `tImageNormalizeProgressiveKind`. Do **not** pass `EXPENSE_RECEIPT` into `siteImageProgressiveUtil.resolveUrls`.
8. **`_w*` pre-gen resize**: Sharp `fit: 'inside'` within `edge×edge` (proportional, no short-edge crop). Never `fit: 'cover'` for variants. Canonical long-side limit also uses `inside`. UI may still use CSS `object-fit: cover` for square frames.

## Typical kind → UI mapping

| UI asset | `epImageNormalizeBindKind` |
|----------|----------------------------|
| Product images | `PRODUCT_IMAGES` |
| News images | `NEWS_IMAGES` |
| Offer header / featured | `OFFER_HEADER_IMAGE` / `OFFER_FEATURED_IMAGES` |
| Preorder header / carousel | `PREORDER_HEADER_IMAGE` / `PREORDER_CAROUSEL_IMAGES` |
| Preset preview | `PRODUCT_PRESET_PREVIEW` |
| Merchant logo / logoRect | `MERCHANT_LOGO` / `MERCHANT_LOGO_RECT` (png) |
| Cart / order note images | `CART_NOTE_IMAGES` / `ORDER_NOTE_IMAGES` |
| Avatar | `USER_AVATAR` |
| Expense receipt | `EXPENSE_RECEIPT` (**only** `s256` + canonical) |
| OpenIM chat image | `OPENIM_CHAT_IMAGE` |
| Print template asset | `PRINT_TEMP_IMAGE` (png) |
| QR / barcode (e.g. storefront wxacode) | `BARCODE` (**only** `s256`+`s512` + canonical; **png**) |

Default full set (most kinds): `_w64/_w128/_w256/_w512` + canonical. Format webp unless logo/printTemp → png.

## Progressive display (~512)

- Thumb / large: **`_w64` + `_w512`** (stored variants; no SIH).
- Larger approx (1024+): **`_w128` + canonical** (large edge `null` → `getContentUrl`).
- Web: `siteImageProgressiveUtil` / `siteImageUtil.getProgressiveUrls(path, 512, kind)`.
- Always pass the **real** kind (news ≠ product). Default `PRODUCT_IMAGES` only when the asset is truly product.

## Code patterns

```ts
// ✅ Sized display
contentUtil.getContentUrlImageForKind(
  env, 'media', // WeChat: 'wechatMedia'
  epImageNormalizeBindKind.PRODUCT_IMAGES,
  path,
  enSiteImageSize.s128,
  enSiteImageSize.s128
);

// ✅ Progressive (web)
siteImageUtil.getProgressiveUrls(path, 512, epImageNormalizeBindKind.NEWS_IMAGES);

// ✅ Expense (only 256)
contentUtil.getContentUrlImageForKind(
  env, 'media',
  epImageNormalizeBindKind.EXPENSE_RECEIPT,
  path,
  enSiteImageSize.s256,
  enSiteImageSize.s256
);

// ✅ Lightbox
contentUtil.getContentUrl(env, 'media', path);

// ❌ No kind / no assert
contentUtil.getContentUrlImage(env, 'media', path, 128, 128);

// ❌ Illegal for expense
getContentUrlImageForKind(..., EXPENSE_RECEIPT, ..., s64, s64);
```

App wrappers (`siteImageUtil.getImageUrl`, CMS helpers, WeChat `getContentUrlImage`) that omit kind are **legacy debt** — new code and refactors must take `kind` and call ForKind (or progressive with kind).

## Changing the policy checklist

- [ ] Update `IMAGE_NORMALIZE_VARIANT_POLICY` in codebase
- [ ] Sync consumer submodules / Lambda policy if separate
- [ ] Backfill or enqueue affected kinds; prune if removing edges
- [ ] Update UI to ForKind / progressive with new edges
- [ ] Confirm `image-normalize-variant-policy.type-test.ts` still typechecks
- [ ] WeChat: align progressive util with web pair rules (no inventing `s1024` mid-tier unless policy adds it)

## Backfill (ops)

```bash
cd xituan_backend
npm run jobs:backfill-normalize -- --env production --dry-run
npm run jobs:backfill-normalize -- --env production --confirm-prod --kinds news_images
# Re-encode `_w*` only (keep canonical): --force --variants-only
npm run jobs:backfill-normalize -- --env production --confirm-prod --force --variants-only --kinds expense_receipt
```

Upload HTTP path uses `imageNormalizeEnqueueUtil` **sync** (wait for terminal). C4 backfill uses **inline worker directly** / async — do not block API.

Requires `platform.async_jobs` migration. See `xituan_agent/devGuide/planned-work/entries/2026-09-image-normalize-full-backfill.md`.

## Review checklist (PR / audit)

- [ ] Every sized URL has an explicit `epImageNormalizeBindKind`
- [ ] Call path is `getContentUrlImageForKind` or progressive util (not bare `getContentUrlImage`)
- [ ] Edges are `enSiteImageSize` values allowed for that kind
- [ ] Preview/lightbox uses canonical URL
- [ ] Domain is `media` / `wechatMedia` for display (not `imagesih`)
- [ ] Policy change did not ship UI before pre-gen

## Related docs

- `xituan_agent/devGuide/media-cdn-sih-domain-split.md`
- `xituan_agent/devGuide/sih-image-size-phase-a.md` (legacy SIH-era notes; prefer this skill + policy util for C3/C4)
- `xituan_agent/devGuide/async-lambda-jobs-framework.md`
- Skill: `cms-form-image-upload` (upload only)
