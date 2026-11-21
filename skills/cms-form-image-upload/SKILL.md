---
name: cms-form-image-upload
description: CMS entity image fields via S3 direct-upload (presign→PUT→complete) then path-only save; Upload fileList + response.path retention. Use when implementing or debugging image fields in editors (products/offers/news/preorder-promotes/settings/presets/compliance).
---

# CMS form image upload (direct-upload + path retention)

## When to use this skill

- Adding image fields to a CMS editor (single image or multiple images).
- Debugging lost merchantId / wrong order / files still going through multipart.
- Aligning with `s3-direct-upload` + `s3-direct-upload-web`.

## Core contract (2026-11)

1. **Do not** append image binaries to entity `FormData` on save.
2. On submit: `cmsDirectImageUploadUtil.materializeFileListPaths` / `materializeSingleFilePath` / `uploadDocument` (`purpose=image|file`, **no-wait** for catalog/logo/compliance).
3. Entity create/update sends **paths only** via `currentImages` / `currentHeaderImage` / logo query keys.
4. Create with images: create text-only → direct-upload with `entityId` → update with paths (same as merchant App).

**Canonical channel = direct-upload.** Multipart `req.files` / `uploadFileToS3` on catalog entity updates is **legacy compatibility only** (old clients that still POST file bytes). New CMS / App / WeChat work must not add multipart image fields; do not treat body string image fields as the path channel when query keys are present. Multipart helpers remain in-repo until old clients are fully retired — not a second first-class upload product.

Helper: `xituan_cms/src/utils/cms-direct-image-upload.util.ts` + transport `direct-upload-web.transport.ts`.

## Frontend pattern (Ant Design Upload)

### 1. State

- `fileList: UploadFile[]` for UI + local `originFileObj` preview (no-wait)
- Persist keys on `file.response.path` (never derive from `file.url`)

### 2. Edit-mode initialization

- For each existing key: `UploadFile` with CDN `url` + `response.path = key`

### 3. onChange

- Keep local File on the row for preview; do not revoke while the form row lives

### 4. Submit

```ts
const paths = await cmsDirectImageUploadUtil.materializeFileListPaths({
  fileList,
  bindKind: epImageNormalizeBindKind.PRODUCT_IMAGES,
  entityId,
});
await productApi.updateProduct(id, formDataWithoutFiles, paths);
```

Compliance docs: `uploadDocument` + `purpose=file`, `entityId` = field name.

Print template fixed images: `uploadOne` with `PRINT_TEMP_IMAGE`, **`waitNormalize: true`**, **`compress: false`** after `convertToGrayscale` (PNG). Do not use catalog no-wait.

## Backend

- Update endpoints already merge `currentImages` / `currentHeaderImage` query; leave `req.files` empty when clients direct-upload.
- **Path channel (direct-upload):** query `currentImages` / `currentHeaderImage` is authoritative. Do **not** prefer FormData body image string fields over those query keys (stale body blocks S3 delete-on-replace).
- **Multipart branch:** only if `req.files` has image parts (legacy client). Keep working for compat; do not extend. After either branch resolves final paths: `removed = old − final` (normalize leading `/`), then `removeCatalogImageKeysFromS3` (expands `_w*` / share JPG siblings).
- Create may return entity without images so client can attach paths on update (offer/preorder softened for this).

## Anti-patterns

- `appendCompressedCmsUploadFiles` / multipart image fields on catalog saves
- Extracting keys from `file.url`
- Persisting `sourceKey` / `upload-incoming/`
- Waiting on normalize for CMS catalog (use no-wait + local File preview)
- Putting catalog image paths into entity `FormData` string fields (`headerImage` / `images`) when query path keys exist
- Gating `materialize*` on `!file.url` (picture-card blob/thumb preview is not a stored path)

