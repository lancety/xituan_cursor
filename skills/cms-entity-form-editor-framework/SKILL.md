---
name: cms-entity-form-editor-framework
description: Standard CMS entity create/update form editor framework (Ant Design Form + FormData), including shared modal pattern for list pages, tempFields cleanup, and safe submission flow. Use when creating or refactoring CMS editors/modals for entities like products, offers, news, preorder-promotes, etc.
---

# CMS entity form editor framework

## When to use this skill

- Creating a new CMS entity editor component (`*Editor.tsx`, `*EditModal.tsx`, `*Form.tsx`).
- Refactoring existing editors to match the standard create/update flow.
- Building list pages that open an editor modal from list items (performance-sensitive).

## Canonical architecture (CMS)

### Component shape (recommended)

- **Editor is responsible for**:
  - Form state (`Form.useForm()`), loading/submitting state, and entity-specific UI fields
  - Translating UI state → `FormData` (via `fieldPreProcessorUtil`)
  - Handling create vs update branching
- **Page/list is responsible for**:
  - Data fetching and list refresh
  - Opening/closing the editor modal (prefer the shared-modal pattern below)

### Shared modal pattern for list pages (performance)

- If the page renders many list items, **do not** render a modal per list item.
- Use a **page-level shared modal** with **split Context** (State vs Actions), so list items only subscribe to actions and do not re-render when modal state changes.

Reference design doc:
- `xituan_agent/devGuide/ListItem-sharedModal-solution.md`

## Standard create/update workflow (FormData)

### Frontend steps

1. **Types first**
   - Use codebase typings (`iXxx`, request typings, propSets) rather than ad-hoc shapes.
   - Avoid `any`. Avoid hardcoding enum values as strings; reference the enum member.

2. **Define core state**
   - `const [form] = Form.useForm();`
   - `const [submitting, setSubmitting] = useState(false);`
   - `const [tempFields] = useState<Set<string>>(new Set());`
   - Optional: `fileList/currentImages` if the entity manages images (see image skill).

3. **Initialize form on edit vs create**
   - In `useEffect`, when `entity` changes:
     - `form.setFieldsValue({...})` for edit
     - `form.resetFields()` + set defaults for create
   - Ensure dependencies include `entity` and `form` (and `language` if multilingual UI depends on it).

4. **Submit flow**
   - `onFinish={handleSubmit}` (or explicit `validateFields()` inside modal footer).
   - Build `cleanedValues` by removing everything in `tempFields`.
   - Convert to `FormData` using:
     - `fieldPreProcessorUtil.preprocessFormDataFields(cleanedValues, xxxPropSets, specialFields?)`
   - Append extra fields not covered by propSets (only when needed).
   - Append files (images) only for new uploads (see image skill).
   - Call API:
     - create: `createXxx(formData)`
     - update: `updateXxx(id, formData, ...)`

### Backend expectations (high-level)

- For routes that use `multer` (`upload.array/fields/none`), **restore Request Context in controller entry** before calling any code that needs `merchantId`.

Reference:
- `xituan_agent/devGuide/request-context-multer-workaround.md`

## Anti-patterns to avoid

- Rendering an editor modal inside every list item (performance + language switch re-render).
- Storing non-entity UI-only temporary fields in FormData (always clean via `tempFields`).
- Manually building FormData field-by-field (prefer `fieldPreProcessorUtil`).
- Using `any` in TypeScript editor code.

