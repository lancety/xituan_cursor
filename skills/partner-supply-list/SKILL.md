---
name: partner-supply-list
description: Partner supply product list and change history—product flag is_partner_supply_item, change log (ADD/REMOVE/BASE_PRICE_CHANGE/GST_INCLUSIVE_CHANGE), pricing via calculateInvoiceDetailItem, barcode preference. Use when modifying supply list logic, product create/update hooks, partner-facing list/history APIs, or CMS product/partner UI for supply list.
---

# Partner supply list: model and standard behavior

## When to use this skill

- Changing how products are included in or removed from the partner supply list (`is_partner_supply_item`).
- Adding or changing the supply list **change history** (table, types, recording).
- Implementing or changing supply list or change-history APIs (admin or partner-access).
- Implementing or changing CMS UI for “供货” (supply) toggle, or partner-facing supply list / history tabs.
- Reusing invoice pricing or barcode logic for the supply list.

## Core model

### Product flag

- **Column**: `merchant.products.is_partner_supply_item` (boolean, default `false`).
- **Meaning**: `true` = product appears in the **generic** partner supply list for all partners; `false` = not on the list.
- Partner-specific differences are **not** stored per product; they come from partner fields: `discountRate`, `barCodePreference`, `isGstFreeCustomisable`.

### Change history table

- **Table**: `merchant.partner_supply_list_changes` (partitioned by `merchant_id`).
- **Purpose**: Global log of “what changed” so partners can see adds, removes, and base-price changes. Per-partner prices are **computed at read time** via the same pricing util.
- **Change types** (enum `epPartnerSupplyListChangeType`):
  - **ADD**: `is_partner_supply_item` changed `false` → `true`.
  - **REMOVE**: `is_partner_supply_item` changed `true` → `false`.
  - **BASE_PRICE_CHANGE**: `base_price` changed while the product is (or was) on the supply list.
  - **GST_INCLUSIVE_CHANGE**: product `is_gst_free` changed while on the supply list (合作商可见：改为含GST / 改为不含GST).
- **Fields**: `merchant_id`, `product_id`, `change_type`, `old_is_partner_supply_item`, `new_is_partner_supply_item`, `old_base_price`, `new_base_price`, `new_is_gst_free` (for GST_INCLUSIVE_CHANGE; old = !new), `changed_at`, optional `operator_id`, `note`.

## Standard operations

### 1. Recording changes (product create/update)

- **Create**: If the new product has `is_partner_supply_item === true`, insert one row with `change_type = 'ADD'`, `new_is_partner_supply_item = true`, `new_base_price = basePrice`.
- **Update**:
  - If `is_partner_supply_item` changes `false` → `true`: insert one row `ADD`.
  - If `is_partner_supply_item` changes `true` → `false`: insert one row `REMOVE`.
  - If `is_partner_supply_item` stays `true` and `base_price` changes: insert one row `BASE_PRICE_CHANGE` with `old_base_price` and `new_base_price`.
  - If `is_partner_supply_item` stays `true` and `is_gst_free` changes: insert one row `GST_INCLUSIVE_CHANGE` with `new_is_gst_free` (old = !new for display).
- **Hook**: In `ProductService` (or equivalent) after create/update, compare old vs new and call `partnerRepository.createSupplyListChange(...)`. Do not block the main flow on failure; log and continue.

### 2. Selecting products for the supply list

- **Base filter**: `merchant_id`, `is_partner_supply_item = true`, `status = 'ACTIVE'`, `deleted_at IS NULL`.
- **Barcode preference** (partner’s `barCodePreference`):
  - **UNIQUE**: Include all products from the base filter.
  - **SHARED**: Include only products that either have no shared barcode (`bar_code_shared IS NULL OR = ''`) or are the “shared root” (`bar_code = bar_code_shared`). Exclude products that use another product’s barcode (`bar_code <> bar_code_shared`).
- **Display barcode**: For each row, use `barCodeShared` when partner prefers SHARED and product has it; otherwise use `barCode`.

### 3. Pricing (single-unit, no totals)

- **Always** use the existing invoice item util:  
  `calculateInvoiceDetailItem(product, partner, 1, isGstRegistered, isPriceInclusiveGst)`.
- **Use for display**: `retailPriceBase` (base ex-GST), `unitPrice` (partner unit ex-GST), `gst` (GST per unit). Do **not** use `amount` or `subtotal` for the list view.
- **Tax context**: `isGstRegistered` and `isPriceInclusiveGst` from merchant finance settings.

### 4. Display and behavior norms

- **List tab**: Columns include product name, category, barcode (text + image), base price, partner wholesale price (ex-GST), GST per unit. Default product image size 128px; provide links for 128/256/512/1024 for download.
- **History tab**: Rows from `partner_supply_list_changes` ordered by `changed_at DESC`. Show time, change type (ADD/REMOVE/价格变更/GST 含·不含 变更: "改为含GST" or "改为不含GST"), product (name + image when available), old/new base price where applicable. Per-partner prices for history can be derived from old/new base price + partner discount in the same way as the list.
- **Barcode rendering**: Reuse the same logic as PrintTemps (e.g. canvas + JS barcode library) so barcodes match invoices/labels.

## APIs and routes

- **Supply list**: `GET .../supply-list/:merchantId/:partnerId` → `{ partner, items: iPartnerSupplyListItem[] }`. Items include `productId`, `name`, `category`, `barCode`, `image` (first image path), `retailPriceBase`, `unitPrice`, `gst`.
- **Change history**: `GET .../supply-list/:merchantId/:partnerId/changes?since=&limit=` → list of change records; for partner-facing UI, enrich with `productName` and `productImage` (first image) per row.
- **Admin**: Same handlers can be mounted under CMS auth. **Partner-facing**: Mount under partner-access (no auth), validate only `merchantId` and `partnerId` as UUIDs; service must ensure `partner.merchantId === merchantId`.

## Types (codebase)

- `iPartnerSupplyListItem`: productId, name, category, metadata, barCode, image, retailPriceBase, unitPrice, gst.
- `iPartnerSupplyListChange`: id, merchantId, productId, changeType, old/new flags and base prices, newIsGstFree (for GST_INCLUSIVE_CHANGE; null as false), changedAt, operatorId, note.
- `iPartnerSupplyListChangeWithProduct`: extends change with `productName`, `productImage` for history tab.

## Reference

- Full design: `xituan_agent/devGuide/Partner-Supply-List-Design.md`.
- Migrations: `xituan_backend/migrations/1710000000238_partner_supply_list.sql`, `1710000000240_partner_supply_list_gst_change.sql`.
- Pricing util: `xituan_codebase/utils/invoiceDetail.util.ts` (`calculateInvoiceDetailItem`).
- Partner types: `xituan_codebase/typing_entity/partner.type.ts` (enums and interfaces above).
