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

## Related docs

- `.cursor/skills/s3-bucket-access/SKILL.md` — private prefix CDN Deny / bucket policy
- `xituan_agent/devGuide/s3/README.md` — prefix registry
