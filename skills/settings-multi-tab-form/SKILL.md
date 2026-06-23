---
name: settings-multi-tab-form
description: Implements current xituan settings pages: CMS merchant settings with one Form and category tabs, plus Platform settings where ORDER uses FormData and PSP_CHECKOUT uses a separate JSON form.
---

# Settings Forms

Use this skill for CMS merchant settings, platform order settings, and platform PSP checkout settings.

## CMS Merchant Settings

CMS settings use one Ant Design `Form` with field names nested by category.

```tsx
name={[epPlatformSettingCategory.OPERATION, 'businessName']}
name={[epPlatformSettingCategory.FINANCE, 'gst', 'isRegistered']}
```

Current pattern:

- Load settings from context/API once and call `forms.setFieldsValue(formValues)` for all categories.
- Keep category-specific upload state for operation logo fields (`currentLogo`, `currentLogoRect`, file lists).
- Save through one `handleSave(category)` branch.
- `OPERATION` uses FormData:
  - remove multilingual temp fields;
  - preprocess with `fieldPreProcessorUtil.preprocessFormDataFields(cleanedValues, operationPropSets)`;
  - append new logo files only;
  - pass retained current logo keys as query/current-image arguments.
- Other merchant categories usually submit JSON, with minimal display/submit normalization.
- Date-only values, such as GST registration date, must be formatted as `YYYY-MM-DD`.

Reference: `xituan_cms/src/pages/settings.tsx`.

## Platform ORDER Settings

Current platform settings page does not use multiple tabs. It loads platform-only settings from context and writes ORDER fields under:

```tsx
name={[epPlatformSettingCategory.ORDER, 'pendingPayExpMinutes_offer']}
```

Current pattern:

- Normalize `orderSettingNumber` fields to numbers before `form.setFieldsValue` so `InputNumber` displays correctly.
- Save ORDER as FormData with numeric keys from `orderSettingNumber`.
- Backend handles ORDER multipart data with `fieldProcessorUtil.processFormDataFields(req.body, orderPropSets)`.
- Backend restores request context if multer dropped it, using `restoreRequestContextFromReq` and `runWithContextAsync`.

References:

- `xituan_platform/src/pages/settings.tsx`
- `xituan_platform/src/lib/api/platform-setting.api.ts`
- `xituan_backend/src/domains/platform-setting/controllers/platform-setting.controller.ts`

## Platform PSP Checkout Settings

`PSP_CHECKOUT` is a separate JSON form on the same platform settings page.

Current pattern:

- Use a separate `pspForm`.
- Normalize/default with `pspCheckoutPlatformSettingsUtil`.
- Submit JSON via `updateSetting(epPlatformSettingCategory.PSP_CHECKOUT, { settings, description })`.
- Do not use FormData for PSP checkout.
- Use enum references such as `epPaymentMethod.WECHAT`, never hardcoded enum strings.

## Shared Rules

- Keep category and enum values referenced through enums (`epPlatformSettingCategory`, `epPaymentMethod`, etc.).
- Use codebase propSets for FormData categories.
- Use `request-context-multer` for backend routes that accept multipart data.
- Do not clear the form before applying refreshed settings unless the local page intentionally resets user edits.
- Keep CSS/theme behavior aligned with `mainlayout-content-padding-rule` and Ant Design custom CSS rules.

## When To Delete Or Split

If a new settings page no longer has categories/tabs and only edits a single JSON object, use a smaller page-specific form pattern instead of forcing this multi-category structure.
