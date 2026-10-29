---
name: merchant-currency
description: >-
  Xituan merchant book currency (currently hardcoded AUD), ledger source
  currency, FX into books, and 计入/不计入 rules. Use when adding merchant currency
  settings, changing epLedgerCurrency, fxToMerchant / fxCnyToAud, ledger intake
  计入账目汇总, auto-posted revenues/expenses, tax overlay conversion, or any
  AUD/CNY/RMB amount path that could conflict with existing books.
---

# Merchant book currency

Read **`xituan_agent/devGuide/merchant-currency/README.md`** before changing currency, FX, or what gets posted to income/expense management. This skill is the agent checklist so a later “merchant currency” feature does not silently rewrite or double-count existing books.

## Frozen as-is (do not invent a parallel model)

| Fact | Current lock |
|------|----------------|
| Merchant book currency | **AUD**, `ledgerSourceCurrencyUtil.defaultMerchant()` — **no** `merchant.currency` column |
| Books amounts (`revenues` / `expenses` / orders / invoices) | **Merchant-currency numbers, no currency column** — today that means AUD |
| Ledger block source currency | `epLedgerCurrency` **AUD \| CNY** only; UI shows CNY as **RMB**; never persist `RMB` |
| 计入账目汇总 | `blockCurrency !== merchantCurrency` |
| 不计入 | Same currency (today AUD–AUD): recon only, **manual** management records |
| Auto books | Counted-in block: ≤1 revenue/expense **per type** if amount > 0 after FX; delete block → cascade cleanup |
| FX for posting books | Stored `fxToMerchant` = merchant units **per 1 source unit**. Intake UI asks source units per 1 merchant unit (e.g. 1 AUD = 4.76 RMB → enter 4.76) and **inverts** before save |
| FX on tax overlay | Separate **FY** `fxCnyToAud` — display/sidecar only; **must not** be a second write into `revenues`/`expenses` |

Timezone (`platform`/`merchant` settings IANA) is **not** currency. Do not reuse timezone as a currency switch.

## Before writing code

- [ ] Read the [authority doc](../../xituan_agent/devGuide/merchant-currency/README.md) §结论 and §实装商户货币时
- [ ] Amounts going into `revenues` / `expenses` are **already in merchant currency** (converted) or still in source currency? Mixing the two in one column is a data conflict
- [ ] Inclusion still uses **inequality of currencies**, not “foreign looks like CNY” hardcoded
- [ ] Counted-in save still requires a **positive** `fxToMerchant`; missing rate → `LEDGER_BLOCK_FX_REQUIRED`
- [ ] Auto-posted rows stay tied by **`ledger_block_id`**; do not also invent a loose `source_id` path for expenses
- [ ] FX counted-in blocks also upsert **one** `LOAN_FROM_DIRECTOR` revenue (not P&L). Never auto-post loan-to. Domestic loan rows are manual revenue/expense only.
- [ ] Tax overlay conversion stays **read-only**; P&L numbers for counted-in blocks come from the auto books
- [ ] Display code: store `epLedgerCurrency.CNY`, show `RMB` via `displayCode` — do not add a third enum value for RMB
- [ ] If adding a real merchant currency column: **plan history** (existing AUD books, existing `fxToMerchant`, tax `fxCnyToAud`) — do not flip `defaultMerchant()` in isolation

## Do not

| Do not | Why |
|--------|-----|
| Treat all Alipay as “always 计入” | 计入 is **≠ merchant currency**. If merchant were CNY, Alipay CNY–CNY would be 不计入 |
| Treat all CBA as “always 不计入” | Same: CBA AUD vs a future non-AUD merchant **would** 计入 |
| Write source-currency totals into `revenues.total_amount` | That column is merchant currency |
| Convert again in tax P&L for blocks that already auto-posted | Double count |
| Persist currency code `RMB` | Enum/CHECK is `CNY` |
| Use tax FY rate to create/update management records | Books use **block** `fxToMerchant` |
| Delete `ledger_block_id` / unique-per-type without a replacement | Block delete and upsert depend on it |
| Rename/replace `cnyToAud` helpers by copying a second converter | Generalize **one** “source × rate → merchant” path |

## Canonical code

| Piece | Path |
|-------|------|
| Merchant default + 计入 test + RMB display | `xituan_codebase/utils/ledger-source-currency.util.ts` |
| Rate parse / multiply | `xituan_codebase/utils/ledger-fx.util.ts` (names still say CnyToAud; semantics = merchant per 1 source) |
| Auto books mapping | `xituan_backend/src/domains/financial/utils/ledger-intake-books.util.ts` |
| Sync on complete/patch/delete | `LedgerIntakeService.syncAccountingBooks` |
| Enum | `epLedgerCurrency` |
| Migration | `1710000000383_ledger_block_accounting_books.sql` |

## Related docs

- `xituan_agent/devGuide/merchant-currency/README.md` — authority
- `.cursor/skills/ledger-intake/SKILL.md` + `xituan_agent/devGuide/ledger-intake/README.md` — 数据块保存/删除；计入公式仍以本文为准
- `.cursor/skills/revenue-expense-idempotency/SKILL.md` + `xituan_agent/devGuide/revenue-expense-idempotency/README.md` — 不计入手工创建幂等（与 `ledger_block_id` 自动账正交）
- `xituan_agent/devGuide/tax-return-report-design.md` — 税表快照；overlay 汇率 ≠ 入账汇率
- `.cursor/rules/date-time-format-standard.mdc` — 日期列；与货币独立
