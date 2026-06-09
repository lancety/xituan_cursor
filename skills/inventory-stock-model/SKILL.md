---
name: inventory-stock-model
description: Summarizes the inventory stock model for products, offer_products, and products_preorderable (stock, reserved_stock, total_stock) and the order-mode flows for reserve/confirm/release. Use whenever modifying inventory-management.service, stock-updater, stock-calculator, or tests touching inventory or total_stock.
---

# Inventory stock model & modes

## When to use this skill

Use this skill whenever:

- Working on `inventory-management.service.ts`, `stock-updater.service.ts`, `stock-calculator.service.ts`, or `inventory.service.ts`.
- Changing how orders affect stock for `REGULAR`, `OFFER`, or `PREORDER` modes.
- Touching tables or entities related to:
  - `merchant.products`
  - `merchant.offer_products`
  - `merchant.products_preorderable`
  - `merchant.product_inventory`
  - `merchant.inventory_locks` or inventory transaction logs.
- Updating tests under `tests/unit/inventory/*` or `tests/integration/inventory/*`, or payment webhook tests that assert inventory behavior.

Always read this skill (and the linked devGuide doc) before changing how `stock`, `reserved_stock`, or `total_stock` are updated.

## Quick model summary

### Core fields (per table)

For `merchant.products`, `merchant.offer_products`, and `merchant.products_preorderable`:

- **stock**
  - Meaning: currently available, sellable stock in this context.
  - Special: `-1` means unlimited stock (no upper bound).
- **reserved_stock**
  - Meaning: quantity that has been reserved (locked for orders) but not yet treated as finally sold.
- **total_stock**
  - Meaning: remaining total stock in this context, excluding already sold units.
  - In the limited-stock case, the invariant is:
    - `stock + reserved_stock = total_stock`

Unlimited stock is represented by `stock = -1` (and total_stock is usually `-1` or a value that the business logic treats as "infinite").

### Separation by order mode

- **REGULAR orders**
  - Consume inventory only from `merchant.products`.
- **OFFER orders**
  - Consume inventory only from `merchant.offer_products`.
- **PREORDER orders**
  - Consume inventory only from `merchant.products_preorderable`.

Cross-table transfers (e.g. from `products` to `offer_products` or `products_preorderable`) are **not handled by per-order logic**. They are handled by the lifecycle of offers/preorders:

- When creating an offer or preorder configuration, inventory is "borrowed" from `products` using SQL like:
  - `UPDATE merchant.products SET stock = stock - N, total_stock = total_stock - N WHERE id = ...;`
- When the offer or preorder campaign ends, separate lifecycle logic should return any leftover inventory from the mode-specific table back to `products`.

Order-level inventory logic **must only operate inside the relevant table for the current mode.**

## Order-level flows (reserve, confirm, release)

These flows are implemented primarily in `InventoryManagementService` and `StockUpdaterService`.

### 1. Reserve on order creation (`reserveStockForOrder`)

Context:

- Called when an order is created to prevent overselling.
- Creates `inventory_locks` rows (with `lockType = 'ORDER_RESERVED'`) for limited stock.

Behavior for limited stock:

- `stock -= quantity`
- `reserved_stock += quantity`
- `total_stock` stays **unchanged**

This keeps the invariant:

- Before reserve: `stock + reserved_stock = total_stock`
- After reserve: `(stock - q) + (reserved_stock + q) = total_stock`

Unlimited stock (`stock = -1`):

- Do **not** change numeric stock counts.
- Either skip lock rows entirely or only record audit/transaction logs.

### 2. Confirm on successful payment (`confirmStockOnPayment`)

Context:

- Called after payment succeeds, via `PaymentHandlerService.updateInventoryOnPayment` or directly in tests.
- Operates on `inventory_locks` with `lockType = 'ORDER_RESERVED'`.

Behavior for limited stock:

- `reserved_stock -= quantity`
- `total_stock -= quantity`
- `stock` stays **unchanged**

Invariant:

- Before confirm: `stock + reserved_stock = total_stock`
- After confirm: `stock + (reserved_stock - q) = total_stock - q`

Implementation details:

- Use `StockUpdaterService.updateStockByMode` with:
  - `stockChange = 0`
  - `reservedChange = -quantity`
  - `totalChange = -quantity`
- After looping locks, delete them with:
  - `inventoryLockRepository.delete({ orderId, lockType: 'ORDER_RESERVED' })`

### 3. Release on cancel/expire (`releaseOrderStock`)

Context:

- Used when an order is cancelled or expired, triggered from:
  - `OrderService` (cancel/delete/admin-update flows)
  - `InventoryCronService` (expired order processing)

Behavior for limited stock:

- `stock += quantity`
- `reserved_stock -= quantity`
- `total_stock` stays **unchanged** (because the units were never truly sold).

Invariant:

- Before release: `stock + reserved_stock = total_stock`
- After release: `(stock + q) + (reserved_stock - q) = total_stock`

Implementation details:

- Use `StockUpdaterService.updateStockByMode` with:
  - `stockChange = +quantity`
  - `reservedChange = -quantity`
  - `totalChange = 0`
- After processing locks, delete them with:
  - `inventoryLockRepository.delete({ orderId, lockType: 'ORDER_RESERVED' })`

Unlimited stock:

- Same principle as reserve/confirm: avoid changing numeric counts; only record logs if needed.

## Payment and refund flows

In addition to lock-based flows, payment and refunds adjust stock in the base or mode-specific tables.

### Payment (`updateInventoryOnPayment`)

- For **REGULAR** items:
  - `updateProductInventory`:
    - `stock -= quantity`
    - `total_stock -= quantity`
- For **OFFER**:
  - `updateOfferStock` → `updateOfferProductStock`:
    - `stock -= quantity`
    - `total_stock -= quantity`
- For **PREORDER**:
  - `updatePreorderStock` → `updatePreorderProductStock`:
    - `stock -= quantity`
    - `total_stock -= quantity`

### Refund (`restoreInventoryOnRefund`)

- For **REGULAR** items:
  - `restoreProductInventory`:
    - `stock += quantity`
    - `total_stock += quantity`
- For **OFFER**:
  - `restoreOfferStock` → `updateOfferProductStock`:
    - `stock += quantity`
    - `total_stock += quantity`
- For **PREORDER**:
  - `restorePreorderStock` → `updatePreorderProductStock`:
    - `stock += quantity`
    - `total_stock += quantity`

## Unlimited stock (`stock = -1`)

Guidelines:

- Treat `stock = -1` as "unlimited" for availability checks.
- Reserve/confirm/release should not change numeric stock fields for unlimited entries.
- It is acceptable (and recommended) to still write transaction logs for auditing, but not to adjust `stock`, `reserved_stock`, or `total_stock`.

## How to apply this skill in changes

When editing inventory-related code:

1. **Identify the table and mode**
   - REGULAR → work with `products`
   - OFFER → work with `offer_products`
   - PREORDER → work with `products_preorderable`
2. **Maintain the invariant for limited stock**
   - Ensure `stock + reserved_stock = total_stock` is preserved by your changes.
   - Reserve: move quantity between `stock` and `reserved_stock`, keep `total_stock` unchanged.
   - Confirm: move quantity from `reserved_stock` into "sold" by reducing `reserved_stock` and `total_stock`.
   - Release: move quantity back from `reserved_stock` to `stock`, keep `total_stock` unchanged.
3. **Do not cross tables in order-level flows**
   - Do not move stock between `products` and `offer_products` or `products_preorderable` inside order-level functions.
   - Cross-table transfers should be handled by offer/preorder lifecycle logic (campaign setup/teardown).
4. **Handle unlimited stock explicitly**
   - Short-circuit availability and skip numeric updates when `stock = -1`.
5. **Sync tests with semantics**
   - Update or add tests under:
     - `tests/unit/inventory/*`
     - `tests/integration/inventory/*`
     - payment-webhook tests
   - Make sure expectations for `stock`, `reservedStock`, and `totalStock` match the rules above.

## Reference

For a more detailed, Chinese-language explanation (with concrete SQL examples and test references), see:

- `xituan_agent/devGuide/inventory-stock-model-and-modes.md`

