---
name: platform-store-iap-subscription
description: >-
  Platform merchant subscription store channel (Apple IAP / Google Play): verify
  purchase, ASN/RTDN webhooks, StoreSubscriptionEntitlementService, originalTransactionId
  occupancy/rebind, merchant app StoreKit purchase freshness, cloudflared sandbox ASN.
  Use when changing apple-iap-*, store-subscription-entitlement, merchant IAP checkout,
  Apple ASN, Google Play billing events, subscription webhooks, or cross-merchant Apple bind.
---

# Platform store IAP subscription (Apple / Google)

Canonical detail (Chinese): **`xituan_agent/devGuide/platform-store-iap-subscription.md`**.

Read this skill **before** changing store-channel subscription code. Do not invent a parallel activate path.

## Scope

| In scope | Out of scope |
|----------|--------------|
| Apple monthly IAP + ASN V2 | Stripe / Omipay wallet recurring (keep existing adapters) |
| Google Play monthly (same entitlement core; RTDN later) | iOS prepaid SKUs (rejected on verify) |
| Cross-merchant bind / rebind on store chain id | Connect order fees / application_fee |

`billing_mode` for store monthly stays **`recurring`**. Never use `applyBulkPeriodExtension` for store renewals.

## Architecture (do not bypass)

```text
Merchant App (StoreKit / Play Billing)
  → POST /api/admin/subscription/verify-apple-iap  (Google verify later)
  → AppleIapVerifyService (JWS + status + bind)
  → StoreSubscriptionEntitlementService.activateMonthly

Apple ASN V2  POST /api/webhooks/apple-iap
Google RTDN   POST /api/webhooks/google-play   (when added)
  → resolve merchant
  → same StoreSubscriptionEntitlementService (grant / cut / grace / auto-renew flags)
```

**One entitlement engine** for store channels. Webhook adapters only decode + route; they must not write period/status with ad-hoc SQL.

## Occupancy (bind) rules — Apple

Chain key = `originalTransactionId` (stored as `merchant_subscriptions.external_subscription_id`).

**Active holder** (exclusive) requires **all**:

- `provider = APPLE`
- `billing_mode = RECURRING`
- `current_period_end >= today`

Prepaid/bulk (even if OTID still on the row) is **not** a holder → another merchant may take over after Apple status allows.

| Action | Rule |
|--------|------|
| verify for merchant B | `findAppleChainHolder` exists and ≠ B → `SUBSCRIPTION_APPLE_IAP_BOUND_TO_OTHER_MERCHANT` |
| Payment history alone | Does **not** permanently block |
| On activate | `nullifyExternalSubscriptionIdByValueExceptMerchant(OTID, B)` then set B’s OTID |
| After activate | best-effort `Set App Account Token` → B’s `merchantId` UUID |
| ASN grant (`SUBSCRIBED` / `DID_RENEW`) | Prefer active holder → else token → else payment hint |
| ASN revoke (`EXPIRED` / `REFUND` / `REVOKE` / grace expired) | Resolve by OTID row even if period ended → `cutAccessNow` |

Util: `apple-iap-merchant-bind.util.ts`.

## Verify (Apple) — required gates

1. Map `productId` via `merchantIapProductMapUtil` — **monthly only**
2. Verify JWS / App Store Server API transaction
3. Reject `revocationDate`
4. Reject `expiresDate <= now` (ms, not date-only alone)
5. `getAllSubscriptionStatuses` must be Active / Billing Retry / Grace (not Expired/Revoked)
6. Occupancy assert (above)
7. `activateMonthly` + best-effort `setAppAccountToken`

Idempotency: completed `subscription_payment_records.external_payment_id = transactionId` for same merchant → already activated. **Do not** short-circuit activate for another merchant’s tx id.

## Client (merchant iOS) — purchase hygiene

File: `xituan_app_merchant/src/utils/iap/merchant-iap-storekit.util.ts`

- `appAccountToken` = current `merchantId` (UUID) at `requestPurchase`
- Accept only **fresh** txs for this tap (`transactionDate` ≥ attempt − skew; not expired)
- Before / when StoreKit redelivers stale cancelled txs: `finishTransaction` + `clearTransactionIOS`, then retry purchase once
- Reconcile poll: `onlyIncludeActiveItemsIOS: true` + freshness filter — never activate from cancelled leftovers
- Finish native transaction **only after** backend `activated: true`
- Surface `TIMEOUT` / `SUBSCRIPTION_APPLE_IAP_BOUND_TO_OTHER_MERCHANT` with Alert (modal covers toast)

## Webhook standard flow

1. Parse body → `signedPayload` (ASN) / RTDN message (Google)
2. Verify signature with store library
3. Map notification type → entitlement op:
   - Grant / renew / resubscribe → `activateMonthly`
   - Auto-renew on/off → `setAutoRenewEnabled` / `Disabled`
   - Billing retry → `markBillingRetry`
   - Expired / refund / revoke / grace expired → `cutAccessNow`
4. Always ACK store with **200** after durable handling when verify succeeded; hard verify failure may 400 (ASN). Prefer not to invent parallel status writers.
5. Sandbox ASN URL (Connect) must point at current tunnel:  
   `https://<host>/api/webhooks/apple-iap`  
   Quick Tunnel host changes every restart — update Connect + keep `cloudflared tunnel --url http://127.0.0.1:3050` running. See `xituan_agent/devGuide/cloudflared-tunnel-setup.md`.

## Key paths

| Area | Path |
|------|------|
| Entitlement core | `xituan_backend/.../store-subscription-entitlement.service.ts` |
| Apple verify | `.../apple-iap-verify.service.ts` |
| Apple ASN | `.../apple-iap-asn.service.ts` + `apple-iap-webhook.controller.ts` |
| Bind | `.../utils/apple-iap-merchant-bind.util.ts` |
| Set token | `.../utils/apple-iap-set-app-account-token.util.ts` |
| Env / CAs | `.../utils/apple-iap-env.util.ts`, `resources/apple-root-cas/` |
| SKU map | `xituan_codebase/.../merchant-iap-product-map.util.ts` |
| Error code | `SUBSCRIPTION_APPLE_IAP_BOUND_TO_OTHER_MERCHANT` |
| Merchant UI | `xituan_app_merchant/.../MerchantIosIapCheckoutModal.tsx` |

## Agent checklist

- [ ] Changes go through `StoreSubscriptionEntitlementService` for store monthly grant/cut
- [ ] Occupancy uses **active holder** definition (not payment history forever-bind)
- [ ] Rebind clears other merchants’ OTID before writing new holder
- [ ] Verify rejects expired Apple status / past `expiresDate`
- [ ] App purchase path cannot promote stale StoreKit txs
- [ ] ASN/RTDN only adapts → same entitlement ops
- [ ] New business error codes + i18n (merchant + codebase enum) together
- [ ] Sandbox ASN URL matches live tunnel when testing renewals

## Related

- Full guide: `xituan_agent/devGuide/platform-store-iap-subscription.md`
- Billing direction (Stripe/Airwallex vs store): `xituan_agent/devGuide/platform-subscription-billing-devplan.md`
- Tunnel: `xituan_agent/devGuide/cloudflared-tunnel-setup.md`
- Business errors: skill `business-error-line-errors`
