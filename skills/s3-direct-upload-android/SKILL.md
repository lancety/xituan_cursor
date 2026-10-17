---
name: s3-direct-upload-android
description: >-
  Android Expo direct upload to an opaque putUrl via expo-file-system
  binary PUT. Covers content:// URIs and why sessionType is iOS-only. Use when
  merchant or customer Android uploads with presign. Read s3-direct-upload first.
  Pair with s3-direct-upload-ios; keep one shared JS transport.
---

# S3 direct upload — Android

Read **s3-direct-upload** first. No earlier doc researched Android alone. Current app uploads are still multipart to the backend. This skill is the PUT transport to use when direct upload is actually implemented.

Use the **same** `File.upload` call as `s3-direct-upload-ios` (`httpMethod: 'PUT'`, `UploadType.BINARY_CONTENT`, matching `Content-Type`). Do not add an Android AWS SDK, OkHttp client, or a second presign route.

## PUT deltas vs iOS

- `sessionType` is an iOS URLSession flag. Android accepts it for API consistency and does not switch session mode. Still pass `sessionType: 'foreground'` from the shared helper so iOS stays correct; do not fork the call.
- Default method is POST. Always pass `PUT`.
- Multipart upload type is forbidden (same as iOS): it is not the presigned body.
- Check `result.status` for 2xx. The promise also resolves for HTTP errors.

## content:// URIs

Pickers often return `content://`. `File.upload` needs a readable file. Copy into the app cache as a `file://` URI, PUT that file, then delete the cache copy. Do not stream the `content://` URI through `FormData` to the backend once this transport exists.

## Cleartext

`usesCleartextTraffic` on the Expo apps is for local API dev. Direct upload stays HTTPS (`amazonaws.com` or `content.*`). Do not point `putUrl` at a cleartext host or at the Metro machine. Host rules: [s3-direct-upload/reference.md](../s3-direct-upload/reference.md).
