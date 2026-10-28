---
name: ledger-intake
description: >-
  Xituan CMS ledger intake (流水入账): analysis blocks, save modes, S3 source/processed
  files, classify rules, auto books on 计入, and tax overlay. Use when changing
  ledger-intake page/API, ledger_analysis_blocks, 保存流水, 数据块, CBA/Alipay/standard
  CSV, processed CSV, 仅保存统计, block delete, or row classify vs block 计入账目汇总.
---

# Ledger intake (流水入账)

Read **`xituan_agent/devGuide/ledger-intake/README.md`** for the product flow and **block management** contract. Currency / FX / what “计入账目汇总” writes into revenues/expenses is **`merchant-currency`** — do not fork those rules here. Private S3 GET is **`s3-bucket-access`**.

## Before writing code

- [ ] Work is **row classify** (待审/排除/类别) or **block management** (save/update/delete/currency/FX/books)? Do not mix the two “计入”
- [ ] Block source currency: CBA locked AUD, Alipay locked CNY (UI RMB); standard selectable, default AUD
- [ ] Same currency as merchant (today AUD–AUD) = **不计入**：no auto books; show **收支数据绑定** column on bank (and commerce only when 不计入 — Alipay 计入 normally hides it); optional **manual** create from a row; **do not** set `ledger_block_id` on those manual rows. Hide the column when 计入. Bank **loan from/to** rows: **关联** existing `LOAN_FROM_DIRECTOR` revenue / `LOAN_TO_DIRECTOR` expense, or **创建** one.
- [ ] Counted-in FX: auto-post expenses plus **one** `LOAN_FROM_DIRECTOR` revenue (commerce expense total, or bank loan-from total). **No** auto loan-to. Delete block cascades auto books.
- [ ] Different currency = **计入**：require `fxToMerchant`; complete/PATCH upserts ≤1 revenue/expense per type; delete block cascades
- [ ] Save UI: button **保存流水**; confirm **取消 / 确认保存**. Save writes block header + processed CSV (no original import file). IndexedDB drafts store classified lines only, not the original CSV. Legacy stats-only cloud blocks cannot restore rows. Do not show `parseEmpty` when analyzed rows already exist.
- [ ] Already has processed CSV: **数据块更新** = processed PUT + PATCH header
- [ ] API must **not** return S3 keys or content/media URLs; signed GET `getUrl` only; backend must **not** proxy file bytes
- [ ] `GET /blocks/:id` still has **no lines**; rows live in IndexedDB or processed CSV
- [ ] Loan actions are **from/to director**, not “repayment”
- [ ] Rule keyword badges on 电商数据处理规则 / 收支款规则 can be dragged onto another group to `updateRule`. All classify groups stay visible even when empty. Bank classify uses saved 收支款规则 only — **not** supplier name/aliases (aliases are OCR). Saving rules always uses batch `POST /ledger-intake/rules` `{ items }` (length 1 included). Keyword result conflicts open a list; each keyword picks its own final result before upsert.
- [ ] **收支 vs 源头（硬性）**：流水「已绑定」= 行上有 `revenueId` / `expenseId`。源头业务单不与流水直接绑定；源头 ID 挂在收入/支出的 `sourceId`（行上可缓存 `orderId`/`summaryId`/`invoiceId` 供创建预填）。详见 README「日期金额匹配」。
- [ ] **两阶段匹配**：有源头的类型先扫源头业务单，命中后再按 `sourceId` 扫收入/支出。有账本 → 绑定；无账本 → 可创建（预填源头）。结算特例见 README 表（确认收款金额在 revenues）。无源头类型（普通支出/董事借款）直接扫账本。
- [ ] Date-amount：**仅不计入**块；跳过待审。触发：导入分析后、行/批量分类后、待审重新识别升类后、整理列表对 pin 行补绑。窗：订单 BEFORE 4w+3d；结算/送货/借给/向董事 AFTER 4w+3d；普通支出 BEFORE 4w+3d（只比 `total_amount`+`expense_date`，不比供应商名）。批 ≤50 且银行日跨度 ≤4w。已有账本 ID 不覆盖。重置待审不跑匹配。Local cache / 保存流水 / 数据块更新 is blocked until organize or reset if 待审 still contains classified results.
- [ ] 持久化：匹配/编辑/IndexedDB/processed CSV **同时**保存源头 ID + 账本 ID。
- [ ] 绑定列 UI（银行）：创建↔查看收入/支出 **只视账本有无**；查看源头 **只视有无可解析源头 ID**（与是否自动匹配命中无关）。待审不创建；`PLATFORM_PAYOUT` 只提示不创建。筛选「全部/已绑定/未绑定」在「批量操作」左侧，并过滤当前列表。电商不加绑定筛选。
- [ ] `GET context` is **rules + partners + bound industry/domain ids only**. Do not dump year-wide orders, settlements, expenses, or loan-from revenues. Client analyze classifies by rules; book linking is `date-amount-matches` only.
- [ ] Bank classify has **no IGNORE**. Non-merchant business movements are only **loan from / loan to director**. Legacy `ignore` rows/rules reopen as REVIEW; do not offer 忽略 on filters, result icons, or rule groups.
- [ ] `epBankRowGroup` = amount sign only (income ≥0 / expense <0). **Never** set REFUND from description `refund`. Outbound refund → expense; inbound refund → revenue with `isRefund` chosen in the books editor — not inferred from bank text. Enum `REFUND` is legacy CSV only.
- [ ] Bank list: batch-migrate row under 已选, same columns as the table. Empty date/amount = current-list min/max; empty group = no group filter; description = keyword filter that live-filters the visible list (IME-safe, all tokens AND). Apply the row’s 处理结果 (+ expense subtype) to matching rows in the **current filter**.
- [ ] Preview filter totals: one badge row. Same currency → source amount only. Counted-in (different currency) → source amount plus merchant amount on the next line inside the same badge (`source × fxToMerchant`); missing FX shows merchant as —. Do not format source CNY/RMB with `A$`. Do not duplicate the filter buttons per currency.
- [ ] Saved-block list summaries (bank + commerce, desktop + mobile) show **merchant currency** (today AUD). Counted-in CNY uses that block’s `fxToMerchant`; missing FX shows —. List API / local-cache items include `fxToMerchant`.

## Block lifecycle (do not invent a third persist path)

| State | Where | Restore edit | Auto books |
|-------|--------|--------------|------------|
| `draft` | IndexedDB only (never Postgres) | Yes (this browser) | No |
| uploaded, no files | Block header only (legacy stats-only) | No — re-import | Yes if 计入 |
| uploaded + files | Header + S3 processed CSV | Yes via signed GET processed | Yes if 计入 |

`persistStatus=draft` is client-only. DB CHECK only allows `uploaded`.

## Do not

| Do not | Why |
|--------|-----|
| Store row arrays in Postgres | Rows were dropped; tax uses header totals |
| Treat overlay FY `fxCnyToAud` as the save rate | Books use per-block `fxToMerchant` |
| Proxy CSV through backend | Presign PUT + signed GET |
| Show S3 keys to the browser | Private prefix + CDN Deny |
| Auto-post books for AUD–AUD | Recon / manual records only |
| Change 计入 formula in this feature | Owned by `merchant-currency` |
| Use “repayment” copy for director loans | Pair is loan **from** / loan **to** director |
| Match settlement by summary total/loss/totalPaid/remaining | Confirm payment amounts live in `revenues` |
| Treat source IDs as「已绑定」or gate「查看源头」on matchState | Bound = book IDs only; view source = resolvable source ID |

## Canonical code

| Piece | Path |
|-------|------|
| CMS page | `xituan_cms/src/pages/financial/ledger-intake.tsx` |
| Local draft | `xituan_cms/src/utils/ledger-intake-local-cache.util.ts` |
| Client PUT | `xituan_cms/src/utils/ledger-intake-s3.util.ts` |
| Analyze | `bank-ledger-analyze.util.ts` / `commerce-ledger-analyze.util.ts` |
| Currency gate | `ledger-source-currency.util.ts` |
| Auto books map | `xituan_backend/src/domains/financial/utils/ledger-intake-books.util.ts` |
| Sync | `LedgerIntakeService.complete` / `patchProcessed` / `deleteBlock` |
| Migrations | `1710000000376` … `0385` (0385 drops `source_key`) |

## Related docs

- `xituan_agent/devGuide/ledger-intake/README.md` — authority for blocks / UI / API
- `.cursor/skills/merchant-currency/SKILL.md` + `xituan_agent/devGuide/merchant-currency/README.md` — 计入 / FX / books columns
- `.cursor/skills/s3-bucket-access/SKILL.md` + `xituan_agent/devGuide/s3/README.md` — private prefix
- `xituan_agent/devGuide/ai/ledger-ai-use-cases.md` — AI envelope
- `xituan_agent/devGuide/tax-return-report-design.md` — tax snapshot
