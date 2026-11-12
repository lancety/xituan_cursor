---
name: s3-direct-upload-ios
description: >-
  iOS Expo direct upload to an opaque putUrl via expo-file-system binary
  PUT (not FormData, not RN fetch of file://). Covers photo-library permission
  and HEIC pitfalls from 2026-10 product-image save failures. Use when merchant
  or customer iOS uploads with presign. Read s3-direct-upload first. Pair with
  s3-direct-upload-android; keep one shared JS transport.
---

# S3 direct upload — iOS

Read **s3-direct-upload** first. Platform PUT only. Customer and merchant share one JS transport (Expo ~57 / `expo-file-system` ~57). Do not add an iOS-only AWS SDK or second presign API.

## Known production pitfalls (2026-10 merchant product images)

These caused **product image save failures** on production iOS before direct-upload. Do not reintroduce them on the multipart path or when wiring S3 PUT.

| Pitfall | Required approach |
|---------|-------------------|
| `fetch` + `FormData` file parts | Unreliable on production iOS. For multipart-to-backend interim: `FileSystem.uploadAsync` (or new `File.upload`) for **binary** bodies; XHR only for field-only FormData. For S3 direct upload: binary PUT only — never wrap the file in FormData. |
| Unstable picker URIs (`ph://`, etc.) | Copy into `cacheDirectory` as `file://` before upload (`normalizeUploadFiles` pattern in `merchant-product.api.ts`). |
| Photo library denied | `ImagePicker.requestMediaLibraryPermissionsAsync()`; if not granted throw a typed error (`PHOTOS_PERMISSION`) and show UI — do not proceed to upload. |
| HEIC/HEIF from Photos | Prefer `preferredAssetRepresentationMode: Compatible` so the library yields JPEG/PNG when possible. If still HEIC: convert or re-encode so **bytes and `Content-Type` match** (`image/jpeg`); do not only rename `.heic` → `.jpg`. |

Canonical pick helper today: `xituan_app_merchant/src/utils/merchant-product-image-pick.util.ts` (`pickIos`). Keep pick logic there (or a shared util); transport only receives a stable `file://` + mime.

## PUT

```ts
import { File, UploadType } from 'expo-file-system';

const result = await new File(fileUri).upload(putUrl, {
  httpMethod: 'PUT',
  uploadType: UploadType.BINARY_CONTENT,
  headers: { 'Content-Type': contentType },
  sessionType: 'foreground',
});
if (result.status < 200 || result.status >= 300) {
  throw new Error(`S3 upload failed (${result.status})`);
}
```

- Default `httpMethod` is **POST**. Always set `PUT` for CloudFront/S3 signed URLs.
- `UploadType.MULTIPART` breaks signed binary bodies. Binary only.
- `sessionType` defaults to `'background'` on iOS. Use `'foreground'` so JS can still call `complete` after PUT.
- `Content-Type` must match what was sent to `presign`.
- `upload()` resolves on any completed HTTP response including 4xx/5xx — check `status`.

## ATS

`putUrl` is HTTPS (`upload.*` or legacy `amazonaws.com`). No ATS/cleartext exception for upload. Host rules: [s3-direct-upload/reference.md](../s3-direct-upload/reference.md).
