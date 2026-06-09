---
name: cms-multilingual-form-editor
description: Multilingual object fields in CMS Ant Design forms (MultilingualInput/tempFields/FormData + backend validation via multilingualUtil). Use when implementing multilingual fields in entity editors (title/description/body/name, etc.) and when handling lang + Accept-Language behavior.
---

# CMS multilingual form editor

## When to use this skill

- Adding or refactoring multilingual fields in CMS entity editors (e.g. product `name`, offer `title/description`, news `title/description/body`).
- Debugging “missing language”, “field becomes string”, “temp fields leaked into FormData”, or validation mismatches between FE/BE.

## Frontend standard

### Use the multilingual components

- Use `MultilingualInput` for simple text fields.
- Use `MultilingualInputMarkdown` for rich markdown fields (e.g. news body).
- Pass `tempFields` into these components so they can register transient field names.

### tempFields cleanup is mandatory

- Before converting values → `FormData`, remove all keys in `tempFields` from the values object.
- Then use `fieldPreProcessorUtil.preprocessFormDataFields(...)` to produce `FormData`.

### Do not reshape multilingual objects manually

- Treat multilingual value as an object type from codebase (`iMultilingualContent` / compatible record).
- Avoid ad-hoc parsing/stringifying outside the shared preprocess util, unless the API explicitly requires it.

## Backend standard

### Language handling

- Read primary language from query: `?lang=...` (default `zh` in many controllers).
- Normalize via `multilingualUtil.normalizeLanguageCode(...)`.
- Use `Accept-Language` header as fallback languages (excluding the primary language).

### Validation

- Use `multilingualUtil.isEmptyForLanguage(value, normalizedLang, fallbackLanguages)` for required multilingual fields.
- Avoid validating multilingual content by checking hardcoded props like `zh_cn` directly; it will diverge from supported languages and fallback behavior.

Reference doc:
- `xituan_agent/devGuide/Multilingual-FormData-Validation-System.md`

## Anti-patterns to avoid

- Forgetting `tempFields` → leaks temp keys into FormData → backend type processor may mis-parse fields.
- Using `any` for multilingual values (loses type safety; increases runtime mismatches).
- Hardcoding enum values as strings when choosing language codes or options; reference enum members/constants if available.

