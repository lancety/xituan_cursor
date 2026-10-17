---
name: s3-direct-upload-ios
description: >-
  iOS Expo direct upload to an opaque putUrl via expo-file-system binary
  PUT (not FormData, not RN fetch of file://). Use when merchant or customer iOS
  uploads images or files with presign. Read s3-direct-upload first. Pair with
  s3-direct-upload-android; keep one shared JS transport.
---

# S3 direct upload — iOS

Read **s3-direct-upload** first. No earlier doc researched iOS alone. Current app uploads are still multipart to the backend. This skill is the PUT transport to use when direct upload is actually implemented.

Customer and merchant are Expo ~57 with `expo-file-system` ~57. One JS transport serves iOS and Android. Do not add an iOS-only presign API or AWS iOS SDK.

## PUT

Use `expo-file-system` `File.upload` (binary), not `fetch` on a `file://` URI and not legacy multipart.

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

- Default `httpMethod` is **POST**. A POST to a presigned PUT URL fails the signature. Always set `PUT`.
- `UploadType.MULTIPART` wraps the body and breaks SigV4. Binary only.
- `sessionType` defaults to `'background'` and applies on iOS. Use `'foreground'` so the JS promise is still alive to call `complete`. A background session can finish after the JS runtime is gone; the presign TTL will not save an unconfirmed object.
- `Content-Type` must match presign.

`upload()` resolves on any completed HTTP response, including 4xx/5xx. Check `status`.

## Photos

Follow `merchant-product-image-pick.util.ts` `pickIos`: `UIImagePickerPreferredAssetRepresentationMode.Compatible`, and if the asset is still HEIC/HEIF, presign and PUT `image/jpeg` bytes, not a renamed HEIC. The label and the bytes must match.

## ATS

`putUrl` is HTTPS (`amazonaws.com` or `content.*`). Do not add an ATS exception or cleartext exception for upload. Host rules: [s3-direct-upload/reference.md](../s3-direct-upload/reference.md).
