---
name: auspost-mpb-api-service
description: >-
  AusPost MyPost Business (MPB) Partner API service standard for Xituan:
  extractable ShippingCore, subject-based connections, platform vs merchant
  credentials. Use when implementing or reviewing MPB connect, labels,
  OAuth, env/platform settings, dispatch API adapters, or future external
  partner-access shipping APIs—in-app or stand-alone service.
---

# AusPost MPB API Service Standard

## Goal

MPB 能力以「Partner 平台出标服务」建模，**不**以「仅 CMS 入驻商户打包台习惯」建模。  
站内履约与远期外部商户 API 共用同一 Core，降低拆成独立服务端的成本。

权威资料：`xituan_agent/auspost/`；产品拍板见 planned plan `csv_mode_retention_0ed474ac.plan.md`。

## Mandatory layers

1. **AuspostMpbShippingCore** — HTTP only: platform creds + merchant token + account-number；封装 organisations / prices / shipments / orders / labels / track / oauth token exchange。  
   **禁止** import 订单、CMS、微信、打包台 UI；禁止写死「必须是 merchant 表某行」。
2. **Connection store** — `subject_type` + `subject_id`（`merchant` | `external_partner`）、加密 token、account-number、status、validated_at。经 `get/put/deleteConnection(subject)` 访问。  
   **禁止**把 token 只塞进无法复用的 CMS-only blob 且无抽象读取口。
3. **App orchestration** — dispatch orchestrator / pack bench / 远期 partner-access BFF：领域 DTO → Core。  
   UI 路径规则（未连显示 CSV / 已连显示 API）**只**活在这层。
4. **Platform config** — `AUSPOST_MPB_*` env（命名稳定）；远期可迁 `platform_settings` 或微服务配置。  
   **禁止**把平台 Key 下发客户端或写入商户明文库。

## Hard rules

- Do not put AusPost HTTP inside `DispatchPackBenchModal` or WeChat-only helpers.
- `iDispatchCarrierAdapter` may wrap Core；**不得**成为唯一持有 MPB 协议知识的地方。
- Phase-1 orders: `payment_method: STORED_PAYMENT` only；Charge Account out unless product reopens.
- OAuth **production-only**（testbed 无）；始终保留 Dashboard 贴 Token。
- Token **never** returned plaintext on GET；logs redact full token.
- External portal / separate deploy out of scope until scheduled；仍禁止增加阻碍抽离的耦合。
- Connection 表若在 `merchant` schema：是否 50 分区以开发者对该表的明确答复为准（当前连接表：**不用**分区）。

## Platform env (do not commit secrets)

```env
AUSPOST_MPB_BASE_URL=
AUSPOST_MPB_PARTNER_ID=
AUSPOST_MPB_API_KEY=
AUSPOST_MPB_API_KEY_PASSWORD=
```

Merchant token：**CMS/App 贴 Token（或 OAuth）→ Connection DB**；Track 轮询按 `merchant_id` 读库。不要用全局 `*_MERCHANT_TOKEN` env。

Headers (lowercase): `authorization` Basic, `auspost-partner-id`, `auspost-merchant-token`, `account-number`（除 Get Organisation / Track）。

## Pack-bench UI (app layer only)

| Connection | Visible paths |
|------------|---------------|
| Not connected | manual + CSV（不渲染 API） |
| Connected | manual + API（不渲染 CSV） |

## Checklist before merge

- [ ] New MPB call lives in Core（或仅被 Core 使用的 thin client）
- [ ] Connection keyed by subject，不是硬编码「仅 merchant_settings 某 category」且无 port
- [ ] Pack-bench / CSV visibility changes do not alter Core signatures
- [ ] Platform env names unchanged or documented migration
- [ ] No order/CMS imports in Core module
- [ ] Secrets redacted in logs；GET status never echoes raw token

## Related（Track 轮询）

- 算法：`xituan_agent/devGuide/delivery/auspost-mpb-track-poll-design.md`
- 定时入口：`backend-internal-cron` skill + `xituan_agent/devGuide/backend-internal-cron-design.md`
- Core `getTrack` 供 tick 调用；客户端禁止直打 AusPost Track
