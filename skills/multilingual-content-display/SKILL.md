---
name: multilingual-content-display
description: Multilingual object (iMultilingualContent) structure, display logic, and codebase utils. Use when reading/displaying multilingual fields from API, handling Accept-Language, or using multilingualUtil. Covers backend processing, consumer apps (wechat/site), and CMS display.
---

# Multilingual Content Display and Utils

## When to use this skill

- Displaying multilingual fields (product name, offer title, news title, etc.) from API responses.
- Implementing language-aware API requests (Accept-Language header).
- Using `multilingualUtil` for getLocalizedText, isMultilingual, isEmptyForLanguage.
- Migrating string fields to iMultilingualContent.
- Backend language middleware and preferred-language resolution.
- Consumer apps (wechat app, xituan_site) language detection and display.

For **CMS form editing** (MultilingualInput, tempFields, FormData), use skill **cms-multilingual-form-editor**.

---

## 1. iMultilingualContent object structure

```typescript
// xituan_codebase/utils/multilingual.type.ts
interface iMultilingualContent {
  intl: true;        // Required: marks as multilingual object
  en?: string;
  zh?: string;       // Generic Chinese fallback
  zh_cn?: string;
  zh_tw?: string;
  [key: string]: string | boolean | undefined;
}
```

- **intl: true** is mandatory; without it, the object is not treated as multilingual.
- Supported languages: `epSupportedLanguages` (en, zh, zh_cn, zh_tw).

---

## 2. multilingualUtil (xituan_codebase/utils/multilingual.util.ts)

### 2.1 Display (read) - getLocalizedText

```typescript
multilingualUtil.getLocalizedText(
  content: iMultilingualContent | string | undefined,
  language: string,           // e.g. 'zh_cn', 'en'
  fallbackLanguages: string[] = []  // e.g. ['zh_cn']
): string
```

- If not multilingual, returns the string or empty string.
- Uses language variants (e.g. zh → zh_cn, zh, zh_tw).
- Fallback order: primary language → fallback languages → system default → any available.

### 2.2 Check if multilingual

```typescript
multilingualUtil.isMultilingual(obj: unknown): obj is iMultilingualContent
// Returns true if obj?.intl === true
```

### 2.3 Validation (backend)

```typescript
multilingualUtil.isEmpty(content): boolean
multilingualUtil.isEmptyForLanguage(content, language, fallbackLanguages): boolean
multilingualUtil.normalizeLanguageCode(language): string  // zh-CN → zh_cn
```

### 2.4 Accept-Language

```typescript
multilingualUtil.parseAcceptLanguage(header: string): Array<{code, quality}>
multilingualUtil.getPreferredLanguage(header: string): string
```

---

## 3. Backend: language handling

### 3.1 Consumer-facing APIs (products, news, preorder-promotes)

- Use **LanguageMiddleware** so `req.preferredLanguages` is set from `Accept-Language`.
- Service processes multilingual fields into a single string per request language.
- Example: `productIntlFields`, `categoryIntlFields` define which fields to process.

### 3.2 Preferred languages from request

```typescript
const acceptLanguage = req.headers['accept-language'] || '';
const preferredLanguages = multilingualUtil.parseAcceptLanguage(acceptLanguage)
  .map(p => multilingualUtil.normalizeLanguageCode(p.code));
// Use first supported as primary; rest as fallbacks.
```

### 3.3 Display logic

- For **display** (consumer): process `fieldNames` with `getLocalizedText(content, primaryLang, fallbackLangs)` → replace object with string.
- For **raw** (CMS edit): return object as-is; CMS uses MultilingualInput.

---

## 4. Consumer apps (wechat app, xituan_site)

### 4.1 Language source

- Wechat app: `languageUtil.getCurrentLanguage()` from storage/system.
- Site: next-intl `locale` or user preference.

### 4.2 API requests

- Send `Accept-Language` header built from current language (e.g. wechat: `generateAcceptLanguageHeader()`).
- Backend resolves language and returns localized strings or raw objects depending on API.

### 4.3 Display in UI

```typescript
// When API returns raw iMultilingualContent
const displayName = multilingualUtil.getLocalizedText(
  product.name,
  getAppLanguage(),  // e.g. 'zh_cn'
  ['zh_cn', 'en']
);
```

- If API returns already-localized strings, use directly.

---

## 5. CMS display (not form editing)

- Use `multilingualUtil.getLocalizedText(value, language, ['zh_cn'])` with CMS `language` from context.
- Example: `NewsList`, `UniversalProductSelector`, orders product tooltip.

---

## 6. Migration: string → iMultilingualContent

Reference: `xituan_agent/devGuide/string-multiligual-migration-guide.md`.

Key steps:

1. DB: VARCHAR → JSONB; migrate existing data to `{ intl: true, zh_cn: oldValue, en: oldValue }`.
2. Types: `string` → `iMultilingualContent`.
3. PropSets: move field from stringSet to objSet.
4. Backend: use `fieldProcessorUtil`; validate with `multilingualUtil.isEmptyForLanguage`.
5. Frontend: MultilingualInput + tempFields (see cms-multilingual-form-editor).
6. Display: replace manual string handling with `getLocalizedText`.

---

## 7. UI text length consistency

- Keep the same UI text in different languages roughly similar in **pixel length** where possible.
- Avoid translations where one language fits the container but another truncates or overflows (e.g. short English vs long Chinese, or vice versa).
- For fixed-width areas (buttons, table columns, card titles), prefer concise translations; consider `-webkit-line-clamp` / ellipsis as fallback when content varies.
- Applies to both i18n UI strings (messages) and multilingual content (names, titles) where copy can be adjusted.

## 8. Anti-patterns

- Do not check `content.zh_cn` directly for validation; use `isEmptyForLanguage`.
- Do not hardcode language codes; use `epSupportedLanguages` or constants.
- Do not manually parse Accept-Language; use `multilingualUtil.parseAcceptLanguage`.
- Do not assume API always returns localized strings; check API contract (raw vs processed).

---

## 9. Reference docs

- `xituan_agent/devGuide/Multilingual-FormData-Validation-System.md` — FormData, validation, tempFields.
- `xituan_agent/devGuide/multilingual-preset-content.md` — Preset content and build-time generation.
- `xituan_agent/docs/multilingual-usage-examples.md` — Intl fields and MultilingualUtil usage.
- `xituan_agent/docs/string-multiligual-migration-guide.md` — String-to-multilingual migration.
- `xituan_codebase/utils/multilingual.util.ts` — multilingualUtil implementation.
- `xituan_codebase/utils/multilingual.type.ts` — iMultilingualContent, epSupportedLanguages.
- Skill **cms-multilingual-form-editor** — CMS form editing (MultilingualInput, tempFields).
