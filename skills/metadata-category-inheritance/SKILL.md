---
name: metadata-category-inheritance
description: >-
  Merchant category ownership versus platform binding, and metadata inheritance
  from the binding root only. Use when changing categories, binding_status,
  platform_domain_id, platform_category_id, parent cascade, metadata scope
  MERCHANT_PARENT_CATEGORY or MERCHANT_SUB_CATEGORY, migration tasks that touch
  binding status, unbind schema copy, or effective metadata merge. Also use when
  the user mentions 分类归属, 平台绑定, 商户父分类, 绑定状态, or metadata 继承.
---

# Metadata category inheritance

Full rules: [`xituan_agent/devGuide/metadata-category-ownership-binding.md`](../../../xituan_agent/devGuide/metadata-category-ownership-binding.md).

A row **owns a platform binding** only when `parent_id` is null and `platform_domain_id` or `platform_category_id` is set. Call that row the binding root. Effective schema walks `parent_id` to that root and reads platform ids only there.

## Do not

- Treat `binding_status <> UNBOUND` as “platform bound”. Migration tasks used to stamp `BOUND_MAPPED` on merchant children and leave `parent_id` unchanged.
- Treat a child’s `platform_domain_id` as a binding. Parent bind must not copy the domain id onto descendants.
- Update `binding_status` from migration create, complete, rollback, or `markCategoryBindingMapped` unless `parent_id IS NULL` and `platform_domain_id IS NOT NULL`.
- Infer the ownership radio from `binding_status`. `parent_id` wins; platform ids are checked only when there is no merchant parent.
- Run unbind schema copy (`COPY_SCHEMA_TO_MERCHANT`) when the row has a merchant parent. Copy only when this row owns a platform binding and the save leaves platform binding.
- Insert unbind copies as `MERCHANT_PARENT_CATEGORY` on a category that still has `parent_id`. The editor lists `MERCHANT_SUB_CATEGORY` only; the hidden parent-scope rows collide on `(merchant_id, scope_type, scope_id, storage_key)` after a later platform bind.
- Count child rows in platform domain or platform category delete guards. Filter `parent_id IS NULL`.
- Assume a merchant child inherits the matching **platform** subcategory’s attributes. Merge uses the binding root’s platform category chain, the root’s merchant attributes, then the child’s own `MERCHANT_SUB_CATEGORY` rows.

## On bind and unbind

- Platform bind clears `parent_id` and flips that category’s attribute scope from sub to parent.
- Parent bind and parent unbind both clear descendants: `platform_domain_id`, `platform_category_id`, and `binding_status = UNBOUND`.
- Saving “hang under merchant parent” or “no ownership” clears this row’s platform ids and sets `UNBOUND`, with no schema copy.
- `MANUAL` and `OVERLAY` rows keep their definition and `origin_tag` when a real unbind copy runs. New platform-layer rows are `FORK_FROM_PLATFORM`. Do not rewrite product jsonb in that copy.

## Before finishing a change

- [ ] Binding-root predicate is the same in backend, CMS, and merchant app.
- [ ] No new write sets a child’s `platform_domain_id` or `binding_status` other than clearing them.
- [ ] Scope flip runs when `parent_id` goes from set to null or the reverse.
- [ ] Dev-only dirty rows are fixed with a one-off against `.env.development`, not a migration, unless the same bad rows exist in production.