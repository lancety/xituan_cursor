---
name: layout-selection
description: Decides which layout to use for xituan CMS pages—MainLayout (authenticated CMS/admin) vs PartnerFacingLayout (third-party partner-facing, no auth). Use when adding new pages, refactoring navigation, or implementing partner-/public-facing views.
---

# Layout selection (xituan CMS)

## When to use this skill

- Adding a new page or route in `xituan_cms`.
- Deciding whether a page should show CMS sidebar, auth, and admin features.
- Implementing views intended for **third-party** users (e.g. partners, suppliers) that must not expose CMS or platform admin.

## Layout types

| Layout | Use case | Auth | CMS features | Typical route |
|--------|-----------|------|--------------|----------------|
| **MainLayout** | CMS staff (merchant admin, managers). All backend management, settings, orders, partners, products. | Required (auth + merchant context). | Sidebar, menu, user/merchant context, permissions. | `/products`, `/partners`, `/orders`, `/settings`, etc. |
| **PartnerFacingLayout** | Third-party partner-facing pages (e.g. supply list viewer). No CMS or admin entry points. | None. | None—header only: merchant logo + basic info. | `/partner-supply-list/[merchantId]/[partnerId]` |

Do **not** use MainLayout for pages that partners or external parties open via a shared link. Use PartnerFacingLayout so no CMS navigation, user menu, or admin features are visible.

## PartnerFacingLayout rules

- **Header only**: long logo (`logoRect` from merchant operation settings), plus merchant display info (businessName, contactName, contactNumber, contactEmail, businessNumber)—aligned with how merchant info appears on invoices.
- **Responsive**: same layout for desktop, tablet, and mobile (single header + content area).
- **No auth**: page and its API are callable without login; access is scoped by URL params (e.g. `merchantId`, `partnerId`).
- **Data**: merchant display for header comes from a dedicated no-auth API (e.g. `GET /api/partner-access/merchant-display/:merchantId`); page content uses partner-access or other no-auth APIs.

## Gate behavior (MerchantStatusGate)

- Routes that start with `/partner-supply-list` are treated as **partner-facing**: allow access even when the user has no merchant or is not logged in (`isPartnerFacingRoute(pathname)`).
- Other routes still follow the existing rules (no merchant → redirect to apply-merchant; merchant not active → merchant-status page).

## Reference

- Design and implementation: `xituan_agent/devGuide/Partner-Supply-List-Design.md`.
- Implemented layout: `xituan_cms/src/components/layout/PartnerFacingLayout.tsx`.
- Implemented gate: `xituan_cms/src/components/MerchantStatusGate.tsx` (partner-facing route allowlist).
