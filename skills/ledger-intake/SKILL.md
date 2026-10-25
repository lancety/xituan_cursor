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
- [ ] Same currency as merchant (today AUD–AUD) = **不计入**：no auto books; show **创建** on both bank and commerce tables; optional **manual** create from a row; **do not** set `ledger_block_id` on those manual rows. Hide the column when 计入. Visibility is by 计入/不计入, not by source kind. Bank **loan from/to** rows: **关联** existing `LOAN_FROM_DIRECTOR` revenue / `LOAN_TO_DIRECTOR` expense, or **创建** one.
- [ ] Counted-in FX: auto-post expenses plus **one** `LOAN_FROM_DIRECTOR` revenue (commerce expense total, or bank loan-from total). **No** auto loan-to. Delete block cascades auto books.
- [ ] Different currency = **计入**：require `fxToMerchant`; complete/PATCH upserts ≤1 revenue/expense per type; delete block cascades
- [ ] Save UI: button **保存流水**; confirm **取消 / 仅保存统计 / 保存统计与文件**. Stats-only cannot 恢复编辑
- [ ] Already has cloud files: **数据块更新** = processed PUT + PATCH header; cannot replace source file
- [ ] API must **not** return S3 keys or content/media URLs; signed GET `getUrl` only; backend must **not** proxy file bytes
- [ ] `GET /blocks/:id` still has **no lines**; rows live in IndexedDB or processed CSV
- [ ] Loan actions are **from/to director**, not “repayment”
- [ ] Rule keyword badges on 电商数据处理规则 / 收支款规则 can be dragged onto another group to `updateRule`. All classify groups stay visible even when empty.

## Block lifecycle (do not invent a third persist path)

| State | Where | Restore edit | Auto books |
|-------|--------|--------------|------------|
| `draft` | IndexedDB only (never Postgres) | Yes (this browser) | No |
| uploaded, no files | Block header only | No — re-import | Yes if 计入 |
| uploaded + files | Header + S3 source/processed | Yes via signed GET processed | Yes if 计入 |

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

## Canonical code

| Piece | Path |
|-------|------|
| CMS page | `xituan_cms/src/pages/financial/ledger-intake.tsx` |
| Local draft | `ledger-intake-local-cache.util.ts` |
| Client PUT | `ledger-intake-s3.util.ts` |
| Analyze | `bank-ledger-analyze.util.ts` / `commerce-ledger-analyze.util.ts` |
| Currency gate | `ledger-source-currency.util.ts` |
| Auto books map | `xituan_backend/src/domains/financial/utils/ledger-intake-books.util.ts` |
| Sync | `LedgerIntakeService.complete` / `patchProcessed` / `deleteBlock` |
| Migrations | `1710000000376` … `0383` (0383 = `ledger_block_id` + `fx_to_merchant`) |

## Related docs

- `xituan_agent/devGuide/ledger-intake/README.md` — authority for blocks / UI / API
- `.cursor/skills/merchant-currency/SKILL.md` + `xituan_agent/devGuide/merchant-currency/README.md` — 计入 / FX / books columns
- `.cursor/skills/s3-bucket-access/SKILL.md` + `xituan_agent/devGuide/s3/README.md` — private prefix
- `xituan_agent/devGuide/ai/ledger-ai-use-cases.md` — AI envelope
- `xituan_agent/devGuide/tax-return-report-design.md` — tax snapshot
