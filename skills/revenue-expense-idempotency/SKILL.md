---
name: revenue-expense-idempotency
description: >-
  Idempotent create for xituan revenues/expenses (management books): bank-row
  source_reference, source_type+source_id dedupe, expense bank_row / loan-from
  expense source. Use when changing createRevenue/createExpense, ledger-intake
  「创建收支」, RevenueEditModal/ExpenseEditModal initial source fields, or
  epRevenueSourceType / epExpenseSourceType / ledger-book-idempotency.util.
---

# Revenue / expense create idempotency

Read **`xituan_agent/devGuide/revenue-expense-idempotency/README.md`** before changing create paths or source keys. Ledger block lifecycle / date-amount match stay in **`ledger-intake`** — do not fork those rules here; only book-create dedupe.

## Before writing code

- [ ] Hit on create = **return existing** book id, not insert a second row
- [ ] Never use amount or date+amount alone as the idempotency key
- [ ] Bank-originated **revenue**: always set `source_reference=bank_row:{clientId}` via `ledgerBookIdempotencyUtil.bankRowSourceReference`; look up reference **before** source 1:1
- [ ] Bank-originated **expense**: `source_type=bank_row` + `source_id=clientId` (migration `0389` columns)
- [ ] 1:1 revenue sources only: `partner_invoice` / `partner_invoice_summary` / `equipment_disposal` / `expense` / `bank_row` — **not** `ORDER` (multi-payment)
- [ ] Private-card advance loan-from: `source_type=expense` + `source_id=expenseId` (one non-refund loan per expense)
- [ ] Manual 不计入 creates must **not** set `ledger_block_id`
- [ ] Preserve stable UUID `clientId` on analyze / CSV restore; do not rely on `index:date:amount`
- [ ] Modals must pass through initial `sourceType` / `sourceId` / `sourceReference` on create (do not overwrite with defaultSourceType only)
- [ ] If changing the key table: edit **devGuide authority** first, then keep ledger-intake README as a **short link**, not a second full copy

## Do not

| Do not | Why |
|--------|-----|
| Dedupe ORDER by `source_id=orderId` alone | Multiple payments per order are valid |
| Rely only on UI hiding「创建」when bound | Unbind / new browser loses row book ids |
| Put full key table only in ledger-intake README | Dual maintenance; authority is this topic’s README |
| Invent a third store for bank line id | Use `source_reference` / expense `source_*` |

## Canonical code

| Piece | Path |
|-------|------|
| Prefix / 1:1 set | `xituan_codebase/utils/ledger-book-idempotency.util.ts` |
| Enums | `epRevenueSourceType`, `epExpenseSourceType` |
| Revenue create | `RevenueService.createRevenue` |
| Expense create | `ExpenseService.createExpense` |
| CMS prefill | `ledger-intake.tsx` `openCreateFromRow` |
| Migration | `1710000000389_expense_source_idempotency.sql` |

## Related docs

- `xituan_agent/devGuide/revenue-expense-idempotency/README.md` — **authority**
- `.cursor/skills/ledger-intake/SKILL.md` + `xituan_agent/devGuide/ledger-intake/README.md` — 流水块 / 匹配 / 绑定；创建幂等摘要链回本文
- `.cursor/skills/merchant-currency/SKILL.md` — 计入自动账 vs 手工账
