---
name: cms-form-image-upload
description: FormData image upload and retention for CMS entity editors (Upload fileList + currentImages S3 keys), multi-tenant merchantId-safe path handling, and correct ordering (old images first, new appended). Use when implementing or debugging image fields in editors (products/offers/news/preorder-promotes/settings, etc.).
---

# CMS form image upload (FormData + S3 key retention)

## When to use this skill

- Adding image fields to a CMS editor (single image or multiple images).
- Debugging issues where a second upload causes old images to “lose merchantId”, disappear, or get deleted.
- Enforcing image list ordering (new images appended to the end).

## Core rule: keep S3 keys, never derive keys from URLs

- Database stores **S3 key** (e.g. `merchantId/product/image/xxx.png`).
- Frontend `currentImages` must contain **S3 keys**, not URLs.
- Do **not** parse `file.url` to reconstruct a key (CDN domains, resize params, and multi-tenant prefixes make this fragile).

## Frontend pattern (Ant Design Upload)

### 1. State

- `fileList: UploadFile[]` for UI
- `currentImages: string[]` (S3 keys) for update retention

### 2. Edit-mode initialization

- For each existing image key, create a `UploadFile` with:
  - `url`: built via `contentUtil.getContentUrlImage(env, 'images', key, ...)`
  - `response.path = key` (store the canonical key here)

### 3. onChange sync

- When Upload changes, recompute `currentImages` only from `file.response.path`:
  - Keep items with `response.path` (existing images)
  - New local files (no url, has `originFileObj`) are uploaded on submit; they should not enter `currentImages` yet

### 4. Submit

- Append only new files into `FormData`:
  - `if (!file.url && file.originFileObj) formData.append(fieldName, file.originFileObj)`
- For updates, send `currentImages` (S3 keys) as query param (JSON string):
  - `?currentImages=[...]` (or entity-specific names like `currentCarouselImages`)

Reference:
- `xituan_agent/devGuide/FormData-Image-Management-System.md`

## Backend pattern (update endpoints)

### Build final image list in stable order

- Keep “old/retained images” first (from query param), in the order provided.
- Append newly uploaded images to the end (from `req.files` uploads).

### Multi-tenant safety

- If old DB images are already in `merchantId/...` form, but `currentImages` arrives missing the prefix (e.g. `product/image/...`), apply a **defensive normalization**:
  - trim leading `/`
  - if missing `merchantId/` and the key looks like an entity path, prefix with current request `merchantId/`
- This prevents accidental S3 deletion due to mismatched key formats.

## Anti-patterns to avoid

- Extracting keys from `file.url` via regex.
- Prepending new uploads before retained images (causes “new images first” ordering).
- Deleting S3 files based on mixed-format keys (always normalize keys consistently before diffing).

