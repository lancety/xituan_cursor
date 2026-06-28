---
name: api-route-groups
description: Defines API route groups in xituan backend—public, CMS/admin, platform, and third-party (partner-access). Use when adding or moving API routes, implementing no-auth endpoints, or designing auth and validation for new roles.
---

# API route groups (xituan backend)

## When to use this skill

- Adding new HTTP endpoints in `xituan_backend`.
- Deciding where to mount a route (prefix, router) and whether it requires auth.
- Implementing **third-party** or **partner-facing** APIs that must not require CMS login.
- Introducing a new route group or auth scheme for a role (e.g. partner, supplier).

## Route groups (overview)

| Group | Prefix | Auth | Context | Purpose |
|-------|--------|------|---------|---------|
| **Public** | e.g. `/api/merchants`, `/api/entityFields`, `/api/merchant-settings`, `/api/platform-settings` | None or optional (e.g. X-Merchant-Id for settings). | No request-context merchant required for some paths. | Default merchant ID, entity fields, platform/merchant settings for apps. |
| **CMS / Admin** | `/api/admin/*` (products, partners, orders, merchant-settings, etc.) | Required: auth + merchant context + optional permission. | `getMerchantId()` from request context; merchantRequiredMiddleware. | All CMS and merchant admin operations. |
| **Platform** | e.g. `/api/admin/users`, platform settings | Required: platform-level auth/role. | Platform scope. | Super-admin, platform config. |
| **Third-party (partner-access)** | `/api/partner-access/*` | **None.** Only validate URL/body params (e.g. UUIDs). | No request context; service receives `merchantId` / `partnerId` from path. | Partner-facing pages (e.g. supply list); future role-specific auth (e.g. PIN) per role. |

Do **not** mix third-party routes with general public routes. Keep them in a dedicated group (e.g. `partner-access`) so that later you can add a single, role-specific auth scheme (e.g. per-partner PIN) without touching public or admin routes.

## Third-party group (partner-access)

- **Mount**: `app.use('/api/partner-access', createPartnerAccessRoutes())` — register before or after other `/api` routes as needed; keep separate from `/api/admin` and public routers.
- **Auth**: No auth middleware. No `getMerchantId()` from context; all scope comes from path.
- **Validation**: Validate only that path params are valid (e.g. `merchantId`, `partnerId` as UUIDs). Reject invalid UUIDs with 400.
- **Service layer**: Use the same service methods as admin (e.g. `PartnerService.getSupplyList(merchantId, partnerId)`). Services must accept explicit `merchantId`/`partnerId` and enforce `partner.merchantId === merchantId` internally.
- **Future**: When adding auth for a role (e.g. partner PIN), add it only to this group (or a sub-router), not to public or admin.

## Example: partner-access routes

- `GET /api/partner-access/supply-list/:merchantId/:partnerId` — supply list (no auth).
- `GET /api/partner-access/supply-list/:merchantId/:partnerId/changes` — change history (no auth).
- `GET /api/partner-access/merchant-display/:merchantId` — merchant display for partner-facing layout header (no auth).

All use UUID validation middleware; no auth headers required.

## Route mount / registration order

Which **group** (this skill) is separate from **where to register** in `app.ts` and route files. Follow rule `.cursor/rules/backend-express-route-order.mdc` when adding or moving routes — especially:

- Register `/api/admin/*` and other CMS paths **before** `createProductRoutes` (public product router runs checkout-capability middleware on all `/api` traffic it receives).
- Inside a router file: static paths and multi-segment paths **before** bare `/:id`.

## Reference

- Implementation: `xituan_backend/src/domains/partner/routes/partner-access.routes.ts`.
- App registration: `xituan_backend/src/app.ts` (search for `partner-access`).
- Mount order rule: `.cursor/rules/backend-express-route-order.mdc`.
- Design: `xituan_agent/devGuide/Partner-Supply-List-Design.md` (§4.4, §8.3).
