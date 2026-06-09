---
name: product-metadata-schema
description: >-
  Xituan product metadata architecture—definitions vs jsonb values, platform industry/domain
  and platform parent/child templates, per-merchant-parent bindings, merged effective schema,
  jsonKey vs platform ref, enums, migration wizard states, schemaVersion and LRU-then-Redis
  cache, and OpenSearch prep. Use when implementing product metadata, category-bound schemas,
  CMS dynamic metadata, print entityFields, validation APIs, migrations, or search facets.
---

# Product metadata schema (xituan)

## Authority and sync

- This skill **mirrors the negotiated Cursor plan** (“商品 Metadata 通用化”) and **chat-locked rules**. When the plan changes, **update this SKILL.md** so agents do not drift.
- **Authoritative product/plan docs** (repo: [`xituan_agent/devGuide/`](../../../xituan_agent/devGuide/) — update when consensus changes):  
  - **Phased delivery**: [`product_metadata_开发计划_b162e071.plan.md`](../../../xituan_agent/devGuide/product_metadata_开发计划_b162e071.plan.md)  
  - **System framework (revised)**: [`商品_metadata_通用化_70835335.plan.md`](../../../xituan_agent/devGuide/商品_metadata_通用化_70835335.plan.md)  
  - **Cursor plan (metadata 退场/替换 + 商户弃用)**: workspace `Metadata 弃用分层整改`（`.cursor/plans/metadata_弃用分层整改_*.plan.md`）— **terminology and merge/suppress rules** below mirror that plan.  
  **Section-level detail** (merge order, `BOUND_NO_MAP`, `platform_category_id`, RBAC, cache deferral) lives in those files; this SKILL summarizes **agent-critical rules** below.
- **Global agent / dev rules checklist** (migrations, TS, dates, API layers, UI, env—**non-metadata-specific**): [`xituan_agent/devGuide/agent-global-dev-rules-checklist.md`](../../../xituan_agent/devGuide/agent-global-dev-rules-checklist.md). Prefer that file for **cross-cutting** items; this SKILL only repeats **metadata-essential** reminders.
- **`value_type` ↔ JSON contract** (Phase 3 doc + shared code path): [`xituan_agent/devStandard/product-metadata-value-type-json-contract.md`](../../../xituan_agent/devStandard/product-metadata-value-type-json-contract.md) + `xituan_codebase` when types/constants are shared.
- **Order line snapshot / search pipeline / error registry / engineering defaults**: [`metadata-order-line-snapshot.md`](../../../xituan_agent/devStandard/metadata-order-line-snapshot.md), [`metadata-search-index-pipeline.md`](../../../xituan_agent/devStandard/metadata-search-index-pipeline.md), [`metadata-product-business-error-registry.md`](../../../xituan_agent/devStandard/metadata-product-business-error-registry.md), [`metadata-phase3-engineering-defaults.md`](../../../xituan_agent/devStandard/metadata-phase3-engineering-defaults.md).
- **Metadata audit + CMS “recent changes”**: **deferred until after Phase 3 core** (see phased plan §Phase 3 已确认决策 extensions).
- Workspace-wide reminders (details in global checklist): **no `any`**, **enum references not string literals**, **date-only `YYYY-MM-DD`**, **English comments in code**.

## When to use this skill

- **`products.metadata`** (jsonb): validation, migration, typing, APIs.
- **Platform** templates: industry → domain; platform parent → child **attribute definitions**.
- **Merchant** categories: parent/child **catalog tree** (`Category.parentId` **only** means merchant merchandising—not “industry”). **In-store** navigation/facets may use this tree; **global** catalog facets use **`platform_category_id`** (+ `domain_id`), not merchant-only names (see framework §1–§2).
- **Binding (snapshot on merchant parent / binding carrier)**: `platform_domain_id`, optional **`platform_category_id`** (platform parent **or** child—single FK), `binding_status`, `mapping_version`. **Wizard/tasks/audit** live in **separate tables** (hybrid model—framework §5.1, phased plan P3-1).
- **CMS**: dynamic metadata form, migration wizard, category metadata admin.
- **Print**: `entityFields` + `metadata.<jsonKey>` paths; preview `sampleData`.
- **Cache**: `schemaVersion` / Redis / ETag—**after Phase 3** per phased plan; until then correctness-first (framework §11).
- **Search later**: keep `ProductSearchService`-style façade and projection-friendly keys.

---

## Two axes (never conflate)

| Axis | Meaning |
|------|---------|
| **Platform templates** | Industry → Domain; Platform parent category → Platform child category. **Default attribute pools** for merchants to bind. |
| **Merchant categories** | Merchant parent ↔ child for **navigation / merchandising**. **`parentId` is not “industry type.”** |

**Per merchant parent**: can bind **different** platform templates than another parent (not one-merchant-one-domain only).

---

## Merge order (effective schema document)

Participation depends on what is bound; default **layer order**:

1. Platform industry (if included via domain binding)
2. Platform domain (if `platform_domain_id` bound)
3. Platform category-chain template attributes (if `platform_category_id` bound—resolve **root → node** on platform tree; framework §4.1)
4. Merchant **parent** category attributes (`MERCHANT_PARENT_CATEGORY` scope)
5. Merchant **sub** category attributes (`MERCHANT_SUB_CATEGORY` scope)

**Conflict policy** (framework §4)

- **Same key may repeat across layers** to express **child overrides parent**; **all definitions for that key in the merge stack must share the same `value_type`**—else **hard error** (e.g. `PRODUCT_METADATA_SCHEMA_TYPE_CONFLICT`), never silent coercion.
- **Platform write-time**: ancestor saves run **descendant closure scan** so a **later parent key** cannot introduce a **type mismatch** with existing child rows (framework §4.3).
- **Published definitions**: **`storage_key` and `value_type` immutable**; change → **new row + migration/wizard** (framework §6).
- **Merchant overlay** of same key only when **types match** (and merge rules allow overlay for non-type fields—document `required` merge in Phase 3).

---

## Schema resolution algorithm (server, single entry)

Input: **`merchantId`**, product **`categoryId`** (leaf subcategory typical).

1. Resolve **leaf category** and **merchant parent** (walk `parent_id` or explicit `main_category_id`—pick one implementation-wide).
2. **Platform base**: read **merchant parent** bindings:
   - If `platform_domain_id` → load merged **industry ∪ domain** attributes.
   - If `platform_category_id` → load attributes for **platform path root → bound node** (parent and/or child per id).
   - If **both** bound → apply **priority merge** above.
3. **Merchant layer**: merge merchant parent then merchant child attribute rows (child adds/overlays per rules).
4. **Ref expand**: rows with **`platform_attribute_id`** take **type / enum / canonical validation** from the platform attribute; **merchant must not change `value_type`**. Display names: respect **`label_locked_to_platform`** vs merchant `display_name` override.
5. Emit **effective schema** for CMS + validator + `entityFields`: each field exposes at least **`jsonKey`**, **`value_type`**, `required`, labels, enum `allowedValues`, **`platformAttributeId`** (optional), and—when sourced from platform rows—**`platformTemplateStatus`** (`ACTIVE` | **`RETIRED`** | **`REPLACED`**) plus **`replacementJsonKey`** when **`REPLACED`** (see **Merchant suppress vs platform template status** below).

**`EXISTS` fallback (merchant definition rows)**

- If **no** merchant attribute rows exist for that scope, API may **project read-only** merged platform template only.
- Once merchant saves definitions for that category path, **`EXISTS` merchant rows** win (fork / customize path).

---

## Values vs keys (critical)

- **`products.metadata`**: object of **`jsonKey` → value** only. **No attribute-definition blobs inside jsonb.**
- **Default alignment to platform**: **do not bulk-rewrite product jsonb keys**. Merchant row uses **`platform_attribute_id`**; effective schema exposes **`jsonKey`** = current physical key in JSON (often equals merchant row `storage_key` in the “physical key = column” model). **Platform canonical `storage_key` and `jsonKey` may differ.**
- **Optional bulk key rename** on products: **exceptional** opt-in job (e.g. legacy print paths)—not default.

### `storage_key` + `value_type` immutability (definition tables)

- **Naming convention (chat-locked)**: `storage_key` must use **camelCase** consistently (e.g. `storageType`, `storageDay`, `cuisineStyle`). Do **not** mix snake_case and camelCase in the same system.
- **Legacy key migration**: if existing snake_case keys exist, migrate via explicit migration task/SQL and standardize on camelCase; do not keep dual naming long-term.
- **`platform_metadata_attribute`**: after **published / in-use** (trigger per implementation—framework §6), **`storage_key` and `value_type` immutable**; new semantics → **new row** + migration.
- **`merchant_metadata_attribute.storage_key`**: immutable after publish when it represents the **physical json key** model; **`value_type`** must stay aligned with platform ref when `platform_attribute_id` is set.
- New semantics → **new row** + optional **data migration**; never mutate published keys/types in place.

---

## Merchant suppress vs platform template status (do not conflate)

| Layer | English (code) | Chinese (product copy) | Meaning |
|-------|----------------|------------------------|---------|
| Merchant task + `merchant.metadata_attribute_suppression` | **suppression** | **弃用** | Record-only: **does not** delete `products.metadata` values or merchant definition rows; tasks can be **rolled back**. |
| Platform row, no replacement | **`RETIRED`** (`epPlatformMetadataTemplateFieldStatus`) | **退场** | Field is sunset **without** a replacement key. **New** product metadata schema for **create** typically **omits** this key (rare); historic jsonb may still exist—merge/validation policy must be explicit in code. |
| Platform row, with replacement | **`REPLACED`** | **替换** | Old + new keys both participate in merged schema; **`replacement_key`** required. CMS/Site use **display priority** (e.g. only-old → show old; new has value → show new only). |

**Naming rules**

- Do **not** call platform **RETIRED/REPLACED** “suppress” or **商户弃用**—that term is for **`metadata_attribute_suppression`** only.
- Merchant table `merchant.metadata_attribute.lifecycle_status` (`ACTIVE` / `DEPRECATED`) is **merchant-side** lifecycle; platform uses **`template_field_status`** (or equivalent column name) **`ACTIVE` | `RETIRED` | `REPLACED`**—keep concepts separate.

**`merchant.metadata_attribute_suppression`**

- Rows mean the merchant **chose** to hide/stop using a key for that scope (migration wizard / deprecate flow).
- **Anti-pattern**: `ProductMetadataSchemaService.computeEffectiveFields` **filter-removing** suppressed keys from `fields` **without** a paired save/validation strategy causes **`PRODUCT_METADATA_INVALID`** when jsonb still contains that key. Prefer **visibility overlay** and/or **save-time strip / allowlist extension** (see Cursor plan §3).

**Site facet**

- **`REPLACED` / `RETIRED`** platform fields default **`facet_in_site = false`** so public list facets do not aggregate retired dimensions; product detail may still render values per schema + display rules.

---

## Platform / merchant tables (minimal mental model)

**`platform_metadata_attribute`**

- `scope_type`: `INDUSTRY` | `DOMAIN` | `PLATFORM_PARENT_CATEGORY` | `PLATFORM_SUB_CATEGORY`
- `scope_id` + **`storage_key`** unique; `value_type`, multilingual `display_name`, `enum_set_id`, `required`, **`template_field_status`** (or equivalent): **`ACTIVE` | `RETIRED` | `REPLACED`**
- **`replacement_key`**: required when **`REPLACED`**; must be **NULL** when **`RETIRED`**

**`merchant_metadata_attribute`**

- `scope_type`: `MERCHANT_PARENT_CATEGORY` | `MERCHANT_SUB_CATEGORY`
- `category_id`, **`storage_key`** (often = physical json key), `platform_attribute_id` (nullable ref), optional **`label_locked_to_platform`**, `value_type`, labels, `enum_set_id`, **`lifecycle_status`** (`ACTIVE` / `DEPRECATED`) — **merchant-side**, not the platform **`RETIRED`/`REPLACED`** enum.

**`merchant_category_template_binding`** (conceptual — prefer **hybrid**)

- **Columns on merchant parent (or carrier)**: `platform_domain_id`, `platform_category_id`, **`binding_status`**, `mapping_version`.
- **Separate tables**: wizard tasks, mapping items, audit links (phased plan P3-1; framework §5.1).

**Enums**

- **`platform_metadata_enum_set` / `_enum_value`** and merchant symmetric tables.
- Product stores **enum `code`**, never enum row UUID.
- **Global vs scoped enum_set**: empty `scope_type`/`scope_id` = reusable global set; non-empty = scoped to industry/domain/platform node to avoid code collisions.
- **Merchant enum reuse**: **reference** platform enum_set (read-only to platform changes) vs **fork** (copy; keep **codes aligned** for migration).
- **ENUM extension**: child may **extend** allowed codes vs default set; **global search facets** expose **default enum only**; **merchant-custom enum channel** stays separate so custom facets are not killed (framework §4.4).

### Platform ENUM `enum_options` vs merchant category metadata (append-only sync)

- **Platform rule**: do **not** rename or delete persisted option `value` codes in JSON `enum_options`. Lifecycle changes use optional per-option `status: epMetadataEnumOptionStatus` — prefer **`DEPRECATED`** only; do **not** change codes in place for published semantics.
- **Active set**: options with **no** `status` or `status === ACTIVE` participate in “new codes” and CMS prefill-from-platform; **`DEPRECATED`** options are excluded from append / default prefill (merchant rows may still retain deprecated entries for historic product values).
- **Merged platform baseline**: `ProductMetadataSchemaService.getMergedPlatformMetadataFields(merchantId, categoryId)` merges **industry → domain → platform category chain** only (same layer order as breakdown), used to diff ENUM codes against merchant `metadata_attribute` rows for the current category scope.
- **List hints**: `GET /admin/products/categories/:id/metadata-attributes` returns each row plus **`has_platform_enum_update`** and **`platform_enum_new_codes_count`** when `value_type = ENUM`, the key exists on the merged platform stack, and the platform **active** code set has values not present on the merchant row (merchant-only extra codes are preserved; sync is append-only).
- **Sync API**: `POST /admin/products/categories/:id/metadata-attributes/:attributeId/sync-enum-from-platform` — append-only merge of missing **non-deprecated** platform options onto the merchant row; returns `{ attribute, appendedCount }` (`appendedCount === 0` when already up to date). Busts merged-schema LRU via `ProductMetadataSchemaService.bustCache()`.
- **CMS**: `xituan_cms/src/pages/category-metadata.tsx` — “平台枚举” column + **同步枚举** when `hasPlatformEnumUpdate`; calls the sync endpoint then reloads the list.
- **Platform CMS UI**: `xituan_platform/src/pages/metadata-attributes.tsx` passes `enumOptionRemovePolicy="platform-no-remove"` into shared `MetadataAttributeFormFields` — ENUM rows **no delete**; per-row **弃用/现行** switch writes `status: DEPRECATED | ACTIVE` on each option (codes stay in JSON).
- **Implementation**: `xituan_backend/src/domains/metadata/utils/category-metadata-enum-sync.util.ts` (`categoryMetadataEnumSyncUtil`).

---

## Binding + migration (hard rules)

**States (`binding_status` or equivalent)**

| State | Meaning |
|-------|---------|
| `UNBOUND` | No platform template binding; merchant-only definitions. |
| `BOUND_NO_MAP` | Bound, migration **not** finished; **product save still allowed**; **global search** must **not** offer **default-enum-dependent** facets until safe (framework §5.3). |
| `BOUND_MAPPED` | Migration complete; normal operations. |

**Hard rules**

1. First bind to a platform template **must open migration wizard**—**no “bind and skip mapping.”**
2. If **manual mapping** or **blocking conflicts** (e.g. **type mismatch**) remain → stay **`BOUND_NO_MAP`** until resolved for strict paths (`entityFields`, merge).
3. While **`BOUND_NO_MAP`**: **do not block product save** by default; **degrade global enum facets** per product decision; **ENUM diffs**: **prompt + auto-map** in Phase 3 (not primary hard-block); **required** mismatches: **Phase 3 must ship explicit merge rule + detection** (phased plan P3-3; framework §5.3).
4. **Merchant category tree rules** when using platform nodes (framework §3): **platform child bind** → **merchant cannot reparent** away from platform-implied parent chain; **platform parent bind** → **that node’s domain locked to default**; children inherit `domain_id`.

**Artifacts**

- **Definition link**: `merchant_metadata_attribute.platform_attribute_id`.
- **Jobs**: `metadata_migration_task` + `metadata_migration_task_item` with `action_type` such as **`LINK_TO_PLATFORM`**, **`RENAME_KEY_AND_LINK`** (optional path), **`DEPRECATE_ONLY`**, **`KEEP_CUSTOM_UNMAPPED`**.
- **Enum map**: `metadata_enum_value_map` or items storing `source_code` → `target_code`.

**Product value migration rules**

- Same key + same type → keep value.
- Different keys + same type → batch rewrite keys **only when user chose that action** (optional compat window).
- Type mismatch → **no auto migrate**.
- Platform adds **required** field → wizard collects **default / block publish** policy.
- Platform sunsets an attribute → use **`RETIRED`** (no replacement) or **`REPLACED`** (with `replacement_key`); merchant historical values may remain in jsonb; do not hard-delete merchant data silently. Do **not** overload the word **弃用** for platform rows—use **退场/替换**.

**Wizard flow (high level)**

dry-run → mapping table (link / optional rename+link / deprecate / keep custom) → enum value map if ENUM → batch job idempotent → **`BOUND_MAPPED`** → optional dual-key read window then cleanup job.

---

## Units

- Store **numbers in smallest canonical base unit** (e.g. grams, mm). **No parallel keys** for kg vs g.
- **Client** (or display-only BFF) converts for display.

---

## Reference type (`VALUE_REFERENCE`) — phase 2 optional

- Value is an **FK id** to another table (cert, media, brand master). **Not** an unvalidated random string id.
- **List APIs**: return raw id + schema marks REFERENCE; **no N+1 expand** by default.
- **Detail**: `expandMetadataRefs=true` or dedicated resolve endpoint; server **batch-hydrates** into e.g. `metadataResolved` read-only block; CMS edits id, display reads resolved.
- **Cache** resolved fragments with bust on target entity change.

---

## Caching and `schemaVersion` (plan + chat)

**Schedule**: **Implement full `schemaVersion` / Redis / ETag / client conditional GET after Phase 3** (phased plan P3-7; framework §11). Until then, optional LRU remains acceptable for dev—do not rely on cross-ECS shared memory.

**Contract**

- **`schemaVersion`** fingerprints the **merged effective schema document** (not SQL DDL, not whole `products` table). Compute on each merge (hash of `mapping_version` + max `updated_at`s or monotonic `effective_schema_revision`).
- Use **`ETag: "<schemaVersion>"`** and **`If-None-Match`** for `GET …/product-metadata-schema`.

**Storage**

- **Key**: `metadataSchema:{merchantId}:{categoryId}` (leaf `categoryId` aligned with resolver).
- **Value**: `{ schemaVersion, fields, computedAt }` (slim fields: prefer validation/render essentials).

**Phased backend**

- **Phase 1 (locked)**: **process LRU only** inside `getEffectiveMetadataSchema`.
- **Later**: **Amazon ElastiCache for Redis** (e.g. `ap-southeast-2`) behind **`REDIS_URL`**; same function, **no Redis URL → LRU**.
- **Local dev**: Docker **`redis:7-alpine`**, `REDIS_URL=redis://127.0.0.1:6379` to test Redis path; omit URL → LRU.
- **ECS**: each task has **its own LRU**; **do not** assume shared memory across tasks—**Redis is the shared tier when needed**.

**Who hits server cache**

- Explicit: `GET …/product-metadata-schema?categoryId=`.
- Implicit: `POST/PATCH` product validation, `GET entityFields/PRODUCT?categoryId=`, optional inline validate on `GET product/:id`.

**Frontend version cache**

- `sessionStorage` key e.g. `metadataSchemaVer:{merchantId}:{categoryId}` storing **`schemaVersion` only**; optional in-memory full fields for long CMS sessions.
- Conditional GET flow: **304** keeps local fields; **200** replaces.

**Operational limits**

- **Lazy populate** only—**no** startup warm-load of all schemas.
- **TTL** (e.g. 5–30 min) as safety net; **correctness** from **write-path bust**.
- **Capacity**: cap fields per category in PRD; LRU **max entry count**; Redis **`maxmemory` + `allkeys-lru`**; slim payload; optional gzip for huge enum payloads; monitor miss rate / memory.

---

## List APIs and `lang`

- Any metadata-aware **sort / coalesce** on list endpoints: pass explicit **`lang`** (same convention as existing product list APIs).

---

## Print templates

- `entityFields` for PRODUCT must include **dynamic** `metadata.<jsonKey>` from merged schema (not only hard-coded food keys).
- **Strict mode (locked)**: on schema merge / resolve errors, **fail explicitly**—**no silent fallback** to static-only fields (framework §10).
- **Preview**: choose leaf category (or representative bound category) to synthesize `sampleData.metadata`.
- **`REPLACED` / `RETIRED`** platform keys: templates bound to `metadata.<jsonKey>` should warn when platform status is non-`ACTIVE`; **`REPLACED`** pairs follow display/preview rules (prefer replacement when both values exist). Keep read behavior explicit in PRD.

---

## Future OpenSearch (not Redis)

- **OpenSearch / Elasticsearch** solves **search / facets / full-text** at scale; **not** schema-version speedups.
- **Prep now**: `ProductSearchService` façade (PG first), **stable facet keys** = `jsonKey` / projection columns, **outbox/events** on product changes, **`product_metadata_search_index`** projection, OpenSearch `_id = productId` (+ `merchantId` filter), **visibility whitelist** for public index docs, **no raw DSL** scattered in handlers.

### Contract lock (must follow now)

1. **Single façade only**: product list + facet read path must go through `ProductSearchService` (or one equivalent façade).  
   - Controllers only parse params / shape response.  
   - No ad-hoc SQL/DSL in controllers/pages.
2. **Stable API shape** (do not break when switching PG -> OpenSearch):  
   - Request core: `page/limit/sort/categoryId/keyword/includeFacets`  
   - Response core: `items/total/page/pageSize/totalPages` + optional `facets[]`  
   - Facet bucket: `{ value: string | null, count: number }`
3. **Stable key contract**: facet key = schema `jsonKey` (`storage_key`), camelCase only, server-side key validation required, and only `STRING/ENUM/NUMBER` are facet-eligible. Fields with **`facet_in_site = false`** (default for **`RETIRED`/`REPLACED`**) must not appear as public facet axes.
4. **Outbox/projection alignment**: metadata/search projection refresh must stay async + idempotent (`product_metadata_search_index_outbox` -> projection). Projection update order must be after metadata migration stable state.
5. **Defer items (do not force now)**: interactive facet filtering, full visibility unified schema, and OpenSearch runtime integration are allowed to be deferred; keep TODOs in backend metadata todo and do not bypass the façade meanwhile.

---

## Review items still open in plan (treat as PRD gates)

Use as **checklist before shipping**—not all are decided; do not invent silent defaults:

- `visibility` model (admin vs site vs print).
- `multilingual` / `date` / `number` JSON shapes vs global date rules.
- Change category with incompatible metadata: block vs mini-wizard.
- Category delete/merge vs in-flight `BOUND_NO_MAP` tasks.
- Migration vs concurrent CMS edits: locking + idempotency.
- Unmapped ENUM codes during compat window.
- When scoped enum_set is mandatory.
- `product_metadata_search_index` update ordering vs migrations.
- **RBAC (locked for P3)**: bind/migrate/metadata defs—**CMS merchant admin/manager only**; platform templates—**platform admin APIs only**—**never** call platform admin routes from CMS clients (framework §9).
- Duplicate product + metadata validation.
- Cross-domain note: order UI reading live `product.metadata` vs snapshot—**define product-facing contract** when touching display.
- **Merchant suppress vs platform `RETIRED`/`REPLACED`**: effective schema + `assertMetadataMatchesSchema` must stay aligned; never assume **suppression** = platform **退场**.

---

## Typical code touchpoints

- Backend: `xituan_backend/src/domains/metadata/services/product-metadata-schema.service.ts` (**`getEffectiveMetadataSchema` / `computeEffectiveFields` / `getMergedPlatformMetadataFields`**, merge + suppression), `category-metadata-enum-sync.util.ts`, `metadata-migration-task.service.ts`, `product-metadata-validation.util.ts`, `xituan_backend/src/domains/product/` (e.g. `admin-product.routes.ts` — `sync-enum-from-platform`) as needed.
- Shared: `xituan_codebase` `metadata.type`, `metadata.enum`, `product.type`, `entityFields.default.ts`, APIs for products / entityFields.
- CMS: `ProductEditModal`, category admin, migration wizard pages.
- Print: `xituan_cms` print temp editor + `entityFields` client.
- Site: `site-product-metadata-schema.util.ts` + product detail metadata rendering (align with CMS **REPLACED** display priority).

---

## Anti-patterns (cause upgrade pain)

- Ad-hoc **`metadata['anything']`** writes without schema validation.
- Duplicated merge logic outside **`getEffectiveMetadataSchema`**.
- Assuming **single-server** memory cache in **multi-ECS** production.
- Using **enum DB ids** inside product jsonb.
- Mixing **weight_kg** and **weight_g** keys.
- **Silent `storage_key` renames** on published definitions.
- Embedding **OpenSearch queries** in random controllers without façade.
- **Skipping migration wizard** when binding platform templates.
- **Removing suppressed keys from merged `fields` with `filter()`** with no save-path compensation → breaks validation when jsonb still has the key.
- Calling platform **退场/替换** “suppression” or reusing **商户弃用** wording for **`RETIRED`/`REPLACED`** rows.
