---
name: ledger-intake
description: >-
  Xituan CMS ledger intake (流水入账): analysis blocks, save modes, S3 source/processed
  files, classify rules, auto books on 计入, and tax overlay. Use when changing
  ledger-intake page/API, ledger_analysis_blocks, 保存流水, 数据块, CBA/Alipay/standard
  CSV, processed CSV, 仅保存统计, block delete, or row classify vs block 计入账目汇总.
---

# Ledger intake (流水入账)

Read **`xituan_agent/devGuide/ledger-intake/README.md`** for the product flow and **block management** contract. Currency / FX / what “计入账目汇总” writes into revenues/expenses is **`merchant-currency`** — do not fork those rules here. Private S3 GET is **`s3-bucket-access`**. Manual「创建收支」dedupe keys are **`revenue-expense-idempotency`** — do not maintain a second full key table here.

## Before writing code

- [ ] Work is **row classify** (待审/排除/类别) or **block management** (save/update/delete/currency/FX/books)? Do not mix the two “计入”
- [ ] Block source currency: CBA locked AUD, Alipay locked CNY (UI RMB); standard selectable, default AUD
- [ ] **Rule (bank + commerce):** same currency = **复查 + 手动创建**（不计入，无自动账）；different currency = **计入**：已分类有金额的类型各 1 条收支 + **loan from/to director 配平**（须 `fxToMerchant`）
- [ ] Same currency (today AUD–AUD) = **不计入**：no auto books; show **收支数据绑定** column on bank (and commerce only when 不计入 — Alipay 计入 normally hides it); optional **manual** create from a row; **do not** set `ledger_block_id` on those manual rows. Hide the column when 计入. Bank **loan from/to** rows: **关联** existing `LOAN_FROM_DIRECTOR` revenue / `LOAN_TO_DIRECTOR` expense, or **创建** one.
- [ ] Different currency = **计入**：one book per classified commercial type (commerce categories; bank `revenueByType` / `expenseByType`); balance — net expense → `LOAN_FROM_DIRECTOR`, net revenue → `LOAN_TO_DIRECTOR` (plus row-classified loans). complete/PATCH upsert; delete cascades. Auto-book dates = import range **last day** (`periodEnd`)
- [ ] complete / 数据块更新 must send `periodStart`/`periodEnd` from line dates (bank `occurredAt`, commerce `paidAt`) so counted-in books land on the source range end day
- [ ] Save UI: button **保存流水**; confirm **取消 / 确认保存**. Save writes block header + processed CSV (no original import file). IndexedDB drafts store classified lines only, not the original CSV. Legacy stats-only cloud blocks cannot restore rows. Do not show `parseEmpty` when analyzed rows already exist.
- [ ] Close preview gate: dirty = current `lines`+currency+fx ≠ last cache/save/open snapshot (or never persisted with rows). Confirm **取消 / 不保存并关闭**. Also gate switching import source or opening another block.
- [ ] Already has processed CSV: **数据块更新** = processed PUT + PATCH header
- [ ] API must **not** return S3 keys or content/media URLs; signed GET `getUrl` only; backend must **not** proxy file bytes
- [ ] `GET /blocks/:id` still has **no lines**; rows live in IndexedDB or processed CSV
- [ ] Loan actions are **from/to director**, not “repayment”
- [ ] Rule keyword badges on 电商数据处理规则 / 收支款规则 can be dragged onto another group to `updateRule`. All classify groups stay visible even when empty. Bank classify uses saved 收支款规则 only — **not** supplier name/aliases (aliases are OCR). Saving rules always uses batch `POST /ledger-intake/rules` `{ items }` (length 1 included). Keyword result conflicts open a list; each keyword picks its own final result before upsert.
- [ ] **收支 vs 源头（硬性）**：流水「已绑定」= 行上有 `revenueId` / `expenseId`。源头业务单不与流水直接绑定；源头 ID 挂在收入/支出的 `sourceId`（行上可缓存 `orderId`/`summaryId`/`invoiceId` 供创建预填）。详见 README「日期金额匹配」。
- [ ] **两阶段匹配**：有源头的类型先扫源头业务单，命中后再按 `sourceId` 扫收入/支出。有账本 → 绑定；无账本 → 可创建（预填源头）。结算特例见 README 表（确认收款金额在 revenues）。无源头类型（普通支出/董事借款）直接扫账本。
- [ ] Date-amount：**仅不计入**块；跳过待审。触发：导入分析后、行/批量分类后、待审重新识别升类后、整理列表对 pin 行补绑。窗：订单 BEFORE 4w+3d（锚点=`createdAt` 下单日；支付方式 cover `cash`+`bank_transfer`；不用 `paidAt`）；结算/送货/借给/向董事 AFTER 4w+3d；普通支出 BEFORE 4w+3d（`receipt_total_amount` 或 `total_amount`，优先单据额；不比供应商名）。批 ≤50 且银行日跨度 ≤4w。已有账本 ID 不覆盖。重置待审不跑匹配。Local cache / 保存流水 / 数据块更新 is blocked until organize or reset if 待审 still contains classified results.
- [ ] 创建幂等：稳定 UUID `clientId`；打开创建时预填 bank_row / `source_reference`。**流水界面新建/批量**须写当前已保存块的 `ledger_block_id`（删块 CASCADE）；**绑定已有账 / OCR / 收支管理手工创建禁止写**。无云端块 id 时禁止创建与批量。完整键表见 **`revenue-expense-idempotency`**。
- [ ] 持久化：匹配/编辑/IndexedDB/processed CSV **同时**保存源头 ID + 账本 ID；支出匹配另存 `expenseSupplierName`（API `supplierName`）供绑定列展示。
- [ ] 绑定列 UI（银行）：未绑定时按行方向显示 **创建收入/创建支出** + **绑定收入/绑定支出**（`createKind`：收入类/`amount≥0`→收入，支出类/`amount<0`→支出，退款组按符号走支出退款或收入退款）；绑定打开对应已有列表写回 `revenueId`/`expenseId`。已绑定只视账本有无显示查看；手工创建成功回调须写回账本 ID（订单有源无账本 → 创建后变查看收入，保留 `orderId`）。已绑定旁 **取消绑定**。支出绑定可带 `expenseSupplierName`。待审不创建/绑定；`PLATFORM_PAYOUT` / `CASH_BANKING`（`isReconOnlyBankAction`）只提示、不创建。筛选「全部/已绑定/未绑定」在批量左侧。电商不加绑定筛选。
- [ ] `GET context` is **rules + partners + bound industry/domain ids only**. Do not dump year-wide orders, settlements, expenses, or loan-from revenues. Client analyze classifies by rules; book linking is `date-amount-matches` only.
- [ ] Orphan bind heal on open bank block: `POST /ledger-intake/book-ids/existing` (ids in → still-existing ids out). Do **not** N× `GET /revenues/:id` / `GET /expenses/:id` for existence. Cap **`LEDGER_EXISTING_BOOK_IDS_LIMIT` (500) per side per request**; client chunks larger sets (do not silent-truncate).
- [ ] Bank classify has **no IGNORE**. Non-merchant business movements are only **loan from / loan to director**. Legacy `ignore` rows/rules reopen as REVIEW; do not offer 忽略 on filters, result icons, or rule groups.
- [ ] `epBankRowGroup` default = amount sign (income ≥0 / expense <0). Row Select or a bank rule `rowGroup=refund` sets **REFUND** (description keywords do not infer refund by themselves). On REFUND: reverse commercial result-icon disables (credit → expense recovery; debit → income reverse); **REVIEW** always on; director loan icons off. Persist group through rematch/CSV. Changing group resets the row to REVIEW and clears book/source bind. If `bookLedgerBlockId` is set, confirm first — confirm deletes that ledger-created book; cancel aborts the group change. Auto-match and bind-existing (no `bookLedgerBlockId`) only clear the row.
- [ ] **退款落账**（与收入/支出管理同符号：正常正 / 退款负，加总自然对冲）：REFUND + 正数 → `ExpenseRefundModal`（`is_refund`，金额存负）；REFUND + 负数 → `RevenueEditModal`（`isRefund`，金额存负，单据可空）。列表金额旁 badge-btn orange「退款」。正常 `ExpenseEditModal` 不承载退款 UI。银行流水正负 ≠ 账本正负。日期金额：非退款组只扫 `is_refund=false` 正数账。退款组另扫 `is_refund=true` 且金额为负（全额/部分退款都是 `is_refund`，不看 `paymentType`）；正金额对同 `expenseType` 支出退款，负金额对同 `revenueType` 收入退款；窗 AFTER；与正数池隔离。待审、设备处置、平台提现、现金收入存现、董事借款不自动匹配。规则处理结果对当前金额不合法时只改退款分组、结果留待审。
- [ ] Bank list: batch-migrate row under 已选, same columns as the table. Empty date/amount = current-list min/max; empty group = no group filter; description = keyword filter that live-filters the visible list (IME-safe, all tokens AND). Apply the row’s 处理结果 (+ expense subtype) to matching rows in the **current filter**.
- [ ] Preview filter totals: one badge row. Same currency → source amount only. Counted-in (different currency) → source amount plus merchant amount on the next line inside the same badge (`source × fxToMerchant`); missing FX shows merchant as —. Do not format source CNY/RMB with `A$`. Do not duplicate the filter buttons per currency.
- [ ] Preview list pagination `showTotal`: left of 共 N 条 show **总正 / 总负 / 总累计** for the **current filtered list** (source currency). Bank uses signed `amount`; commerce signs by `direction` (income +, expense −).
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
| Treat tax overlay as a second FX source | Books and overlay both use per-block `fxToMerchant` |
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
| Migrations | `1710000000376` … `0385` (0385 drops `source_key`); expense source cols `0389`; expense refund `0390`; book breakdown `0398` (`book_breakdown` jsonb) |

## Related docs

- `xituan_agent/devGuide/ledger-intake/README.md` — authority for blocks / UI / API
- `.cursor/skills/revenue-expense-idempotency/SKILL.md` + `xituan_agent/devGuide/revenue-expense-idempotency/README.md` — 创建收支幂等（权威）
- `.cursor/skills/merchant-currency/SKILL.md` + `xituan_agent/devGuide/merchant-currency/README.md` — 计入 / FX / books columns
- `.cursor/skills/s3-bucket-access/SKILL.md` + `xituan_agent/devGuide/s3/README.md` — private prefix
- `xituan_agent/devGuide/ai/ledger-ai-use-cases.md` — AI envelope
- `xituan_agent/devGuide/tax-return-report-design.md` — tax snapshot
