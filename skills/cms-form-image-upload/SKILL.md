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

## Backend

- Update endpoints already merge `currentImages` query; leave `req.files` empty when clients direct-upload.
- Create may return entity without images so client can attach paths on update (offer/preorder softened for this).

## Anti-patterns

- `appendCompressedCmsUploadFiles` / multipart image fields on catalog saves
- Extracting keys from `file.url`
- Persisting `sourceKey` / `upload-incoming/`
- Waiting on normalize for CMS catalog (use no-wait + local File preview)

