---
name: entity-form-formdata
description: Implements create/edit forms for entities using FormData, propSets, fieldPreProcessorUtil, fieldProcessorUtil, image retention, multilingual tempFields, and current xituan date handling. Use when adding or refactoring entity editors with multipart/form-data, uploads, multilingual fields, or propSets.
---

# Entity Form + FormData

Use this pattern for single-entity create/edit editors such as Product, News, Offer, Preorder Promotes, Partner, Store, Supplier, and similar CMS/domain entities.

## Current Pattern

- Use codebase typings and propSets (`productPropSets`, `offerPropSets`, etc.) from `xituan_codebase/typing_entity`.
- Frontend preprocesses normal fields with `fieldPreProcessorUtil.preprocessFormDataFields(cleanedValues, entityPropSets, specialFields?)`.
- Backend parses multipart body with `fieldProcessorUtil.processFormDataFields(req.body, entityPropSets, specialFields?)`.
- Do not manually build every entity field into FormData unless the current implementation intentionally does so for a narrow category.
- Do not set the `Content-Type` header when sending FormData; the browser must set the multipart boundary.

## Frontend Flow

1. Use `Form.useForm()`.
2. If multilingual fields exist, keep `const [tempFields] = useState<Set<string>>(new Set())`.
3. If images exist, keep both Upload `fileList` for UI and `currentImages` / `currentLogo` / similar state for existing S3 keys.
4. On edit, `form.setFieldsValue(...)` with API values normalized for controls:
   - number fields become `number` for `InputNumber`;
   - multilingual objects are passed as objects;
   - existing image keys become Upload files with display URL and `response.path = key`.
5. On create, `form.resetFields()`, clear file/image state, and set default object values where needed.
6. On submit:
   - validate fields;
   - copy values and remove every key in `tempFields`;
   - preprocess with `fieldPreProcessorUtil`;
   - append only new files (`!file.url && file.originFileObj`);
   - pass retained existing image keys as query/current-image arguments, not as file fields.

## Image Retention

- Database image values are S3 keys, not CDN URLs.
- `currentImages` must contain stored keys from the API.
- Store the key on Upload items as `response.path`; never parse `file.url` to reconstruct the key.
- On update, backend builds the final image list as retained old keys first, then newly uploaded keys.
- Backend may defensively normalize retained keys with the current `merchantId` before diffing/deleting storage objects.

## Multilingual Fields

- Use `MultilingualInput` / `MultilingualInputMarkdown` and pass `tempFields`.
- Entity propSets should put multilingual object fields in `obj`.
- Backend validation should use `multilingualUtil` and normalized language/fallback behavior, not hardcoded language property checks.

## Date Handling

- Follow the workspace date rule first.
- For date-only database columns (`date` type), transmit `YYYY-MM-DD` strings and do not convert through timezone offsets.
- Current `fieldPreProcessorUtil` converts `sets.date` values to ISO strings and backend parses them to `Date`; use that only for existing datetime-oriented FormData flows, or after explicitly normalizing date-only fields to the expected string contract.
- For timezone datetime fields, convert business timezone to UTC before preprocessing or append the UTC ISO value.

## Backend Flow

- Routes that receive multipart data must use the correct multer middleware:
  - `upload.array(...)` / `upload.fields(...)` for files;
  - `upload.none()` for multipart fields without files.
- If the handler or downstream service uses request context, restore context after multer before doing business work. Use the `request-context-multer` skill.
- Parse fields with `fieldProcessorUtil.processFormDataFields`.
- Merge uploaded files and retained current image keys after field parsing.
- Validate required multilingual fields, file counts, and business rules before persistence.

## References

- `xituan_agent/devGuide/FormData-Image-Management-System.md`
- `xituan_agent/devGuide/Multilingual-FormData-Validation-System.md`
- `xituan_codebase/utils/form.fieldPreProcessor.util.ts`
- `xituan_codebase/utils/form.fieldProcessor.util.ts`
