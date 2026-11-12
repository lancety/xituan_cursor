---
name: s3-direct-upload-android
description: >-
  Android Expo direct upload to an opaque putUrl via expo-file-system binary
  PUT. Covers content:// URIs, expo-image-picker ActivityResultLauncher crashes
  (use DocumentPicker for product gallery), and FormData fetch pitfalls from
  2026-10. Use when merchant or customer Android uploads with presign. Read
  s3-direct-upload first. Pair with s3-direct-upload-ios; keep one shared JS
  transport.
---

# S3 direct upload — Android

Read **s3-direct-upload** first. Use the **same** `File.upload` call as `s3-direct-upload-ios` (`httpMethod: 'PUT'`, `UploadType.BINARY_CONTENT`, matching `Content-Type`). One shared JS transport with iOS. Do not add an Android AWS SDK or a second presign route.

## Known production pitfalls (2026-10 merchant product images)

| Pitfall | Required approach |
|---------|-------------------|
| `expo-image-picker` / `launchImageLibraryAsync` on Android | Caused **ActivityResultLauncher crashes** in production. Product gallery pick uses **`expo-document-picker`** (`DocumentPicker.getDocumentAsync` with `type: 'image/*'`, `copyToCacheDirectory: true`) instead. See `merchant-product-image-pick.util.ts` `pickAndroid`. Do not switch product (or similar multi-image) pick back to `expo-image-picker` without a proven fix. |
| `fetch` + `FormData` file parts | Same as iOS: unreliable on production. Direct upload = binary PUT. Interim multipart to backend: `uploadAsync` / `File.upload` for binaries, not RN `fetch` FormData files. |
| `content://` URIs | `File.upload` needs a readable `file://`. Copy into app cache, PUT, then delete the cache copy (`copyToCacheDirectory` on DocumentPicker already helps; transport still normalizes). |

## PUT deltas vs iOS

- `sessionType` is iOS-only; Android ignores the session mode but shared helpers still pass `'foreground'`.
- Default method is POST — always pass `PUT`.
- No multipart upload type for signed PUT bodies.
- Check `result.status` for 2xx.

## Cleartext

`usesCleartextTraffic` is for local API only. Direct upload stays HTTPS. Host rules: [s3-direct-upload/reference.md](../s3-direct-upload/reference.md).
