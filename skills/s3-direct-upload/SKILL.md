---
name: s3-direct-upload
description: >-
  Unified S3 direct-upload contract for xituan: presign, PUT the opaque putUrl,
  then complete. PUT host is the upload CloudFront distribution (upload-incoming/
  prefix, CloudFront-signed URL). Use when adding or changing browser, iOS,
  Android, WeChat, CMS, Site, or Expo uploads that should skip backend multipart.
  Read reference.md for the live distribution IDs, then the matching platform skill.
---

# S3 direct upload (unified contract)

Authority for the **checklist**. Host, security constraints, and why there is one PUT implementation: [reference.md](reference.md). Platform skills only define how that platform performs the PUT.

Framework (signer, routes, codebase `directUploadUtil`, platform transports) is live — see reference.md. Catalog/OpenIM still need purpose + UI. Expense OCR and OTA stay on presigned `amazonaws.com`.

**Tests:** offline Jest under `xituan_backend/tests` (`--testPathPattern=direct-upload`); live AWS smoke via `npx ts-node --transpile-only scripts/probe-direct-upload.ts` (not in default Jest).

## Only legal sequence

1. Backend `presign` → `{ putUrl, sourceKey }` (plus bind fields the API defines).
2. Client **PUT** the file bytes to `putUrl`. `putUrl` is opaque. Do not parse, rewrite, or swap its host.
3. Backend `complete` → **canonical** path/URL only (images: after sync IMAGE_NORMALIZE).
4. Business save sends that canonical text. No file bytes.

Forbidden:

- Persisting `sourceKey` or the original `.jpg`/`.png`/`.heic` before `complete` succeeds.
- `if (web | ios | android | wechat)` that signs a different host or calls a different presign API.
- Enabling PUT on `media` or `content`. Upload writes go to the upload distribution only. See reference.md.
- Rewriting an S3 SigV4 host to `content` or `media`.
- A second presign/PUT stack split by image vs file, or by OS.
- A second copy of the three-step state machine in a page. Orchestration belongs in `xituan_codebase`; each app supplies a transport.
- Serving **private** prefixes via content/media GET. Who may GET is **`s3-bucket-access`** (ledger-intake and later private keys).

## Presign signing (backend)

Shipped signer for expense/OTA: `s3UploadManager.getSignedPutUrl` (path-style `s3-{region}.amazonaws.com`). Checksum rules that those PUTs need are in reference.md. The upload distribution `putUrl` is a CloudFront signed URL; do not reuse `getSignedPutUrl` and swap the host.

## What each platform skill owns

| Skill | Owns |
|-------|------|
| `s3-direct-upload-web` | Browser `fetch` PUT (shipped expense pattern) |
| `s3-direct-upload-ios` | Expo file PUT on iOS |
| `s3-direct-upload-android` | Expo file PUT on Android |

Customer and merchant Expo apps share **one** JS transport. The iOS and Android skills are native edge rules for that transport, not two upload products.

## Pitfalls (learned in migration)

### Backend mount: never blanket-auth a router on `/api`

Customer routes live at `POST /api/direct-upload/presign|complete` via
`app.use('/api', createCustomerDirectUploadRoutes(...))`.

**Do not** `router.use(authMiddleware.authenticate)` on that router. Express runs
router-level middleware for every request that enters the mount, including
public GETs that should fall through to later `/api` routers
(e.g. `GET /products-preorderable/list` → `TOKEN_REQUIRED` / Merchant ID errors).

Auth must be **per-route** on customer direct-upload only. Admin mount under
`/api/admin` may keep `router.use(authenticate)` because that prefix is admin-only.

### Double `/api` path

| App | Typical `NEXT_PUBLIC_API_URL` | Transport path |
|-----|------------------------------|----------------|
| **Site** | already ends with `/api` | `direct-upload/presign` (no extra `/api`) |
| **CMS** | often host only (`http://…:3050`) | `/api/admin/direct-upload/presign` |

Wrong Site join → `…/api/api/direct-upload/presign` → misses the real route →
falls into product `requestContext` → **Merchant ID is required**.

### complete vs PUT

- `presign` / `complete` = JSON to backend (POST).
- File bytes = **PUT** to opaque `putUrl` (upload CF), not POST to backend.
- Local WeChat/devtools `http://tmp/…` preview is **not** the S3 PUT.

### waitNormalize — when to wait vs no-wait

| Mode | Use when |
|------|----------|
| **wait** (`true`) | Next screen **must** have canonical / `_w*` immediately, or many views only accept CDN URL with no fallback (avatar, OpenIM, print, WeChat notes without local copy). |
| **no-wait** (`false`) | User can leave the upload step before Lambda finishes, **and** display has a safe fallback (below). |

### no-wait display standard (required)

After `complete` with `waitNormalize: false`:

1. Persist **`path || promotedKey`** (never `sourceKey` / `upload-incoming/`).
2. Promote has already copied the object to the business key — **original** is readable on media/content CDN.
3. `_w*` / SIH variants often **do not exist yet** → first paint of sized thumbs 404s.

**Standard UI contract for no-wait surfaces (Site cart notes shipped as reference):**

```text
preferred src = sized URL (_w64 / getImageUrl / getImageUrlForKind)
onError     → swap to original object URL (getOriginalUrl / getPreviewImageUrl)
lightbox    → original URL (already no _w*)
```

- Do **not** block add-to-cart / navigate on IMAGE_NORMALIZE just to avoid thumb 404.
- After normalize completion rewrites DB path to canonical, sized thumbs work on next load.
- If the surface **cannot** implement onError→original (or local File preview), use **wait** instead.

Shipped example: `xituan_site` cart `CartItemCard` / `CartItemRow` + `siteImageUtil.getCartNoteThumbUrl`.

## Related docs

- `.cursor/skills/s3-bucket-access/SKILL.md` — private prefix CDN Deny / bucket policy
- `xituan_agent/devGuide/s3/README.md` — prefix registry
- Web-specific auth/base URL: `s3-direct-upload-web`
