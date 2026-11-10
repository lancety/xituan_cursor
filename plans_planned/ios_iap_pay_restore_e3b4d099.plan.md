---
name: iOS IAP pay restore
overview: 【planned / mid】Merchant iOS 上 IAP 时恢复订阅购买 UI/原生能力；勿再开 Stripe 订阅结账。
todos:
  - id: write-entry
    content: 新增 planned-work entry 2026-11-merchant-ios-iap-pay-restore.md（恢复清单 + 触发）
    status: completed
  - id: update-registry
    content: registry.md Active 增加 merchant-ios-iap-pay-restore，priority mid
    status: completed
isProject: false
---

# Merchant iOS：IAP 时恢复 pay 组件（planned mid）

Cursor plan（planned）：`d:/projects/xituan_module/.cursor/plans_planned/ios_iap_pay_restore_e3b4d099.plan.md`  
台账：`merchant-ios-iap-pay-restore`。

## 台账要点

- **ID**: `merchant-ios-iap-pay-restore`
- **状态**: `planned`
- **优先级**: **mid**（iOS）
- **触发**: 决定在 Merchant App 上架 **App Store In-App Purchase** 订阅购买时

## Entry 必记恢复清单（相对现状）

现状：[`merchant-subscription-commerce.util.ts`](xituan_app_merchant/src/utils/merchant-subscription-commerce.util.ts) 上 `allowsInAppPurchase()` 在 iOS 为 false；订阅页/首页购买与外链账单已按 3.1.1 关掉。审核还看到 PassKit（`@stripe/stripe-react-native` + `merchantIdentifier`）。后续若 iOS 构建再去掉 Stripe plugin，恢复时一并按本清单加回。

### Native / 构建

- [`xituan_app_merchant/app.json`](xituan_app_merchant/app.json)：`@stripe/stripe-react-native` plugin、`merchantIdentifier`（`merchant.au.com.xituan.merchant`）、Google Pay 仅 Android
- Apple Developer / Xcode：Apple Pay capability、Merchant ID、必要时 StoreKit / IAP products（**IAP 路径与 Stripe 钱包路径需产品决策；默认目标是 IAP 收订阅款，Stripe 不再用于 iOS 订阅结账**）
- `package.json` 依赖：`@stripe/stripe-react-native`（若 iOS 完全不用 Stripe 结账，可仅 Android 条件链；IAP 用 StoreKit/expo-iap 等另列）

### 业务开关与 UI

- [`merchant-subscription-commerce.util.ts`](xituan_app_merchant/src/utils/merchant-subscription-commerce.util.ts)：重做 `allowsInAppPurchase()`（iOS 改为 IAP 可用，而非再开 Stripe）
- [`SubscriptionPlansScreen.tsx`](xituan_app_merchant/src/screens/SubscriptionPlansScreen.tsx)：购买/换计划/续订 CTA（今日被 `purchaseEnabled` 关掉）
- [`MerchantBulkPrepaidCheckoutModal.tsx`](xituan_app_merchant/src/components/subscription/MerchantBulkPrepaidCheckoutModal.tsx)：预付/批量结账 UI（今日 Android Stripe；iOS IAP 需替换支付呈现）
- [`MerchantHomeScreen.tsx`](xituan_app_merchant/src/screens/MerchantHomeScreen.tsx)：订阅卡购买/管理入口与外链账单（今日 iOS 已挡）
- i18n：`subscription.hint*Ios` 等「本版不可购」文案 → 改为 IAP 流程文案

### 支付实现（恢复时改写，勿原样开回 Stripe-on-iOS）

- [`merchant-stripe-subscription-pay.util.ts`](xituan_app_merchant/src/utils/merchant-stripe-subscription-pay.util.ts) + [`stripe-merchant-app.enum.ts`](xituan_app_merchant/src/utils/stripe-merchant-app.enum.ts)：Android 可继续 Stripe；**iOS 订阅应走 IAP**，不要为过审再把 Stripe Apple Pay 加回订阅结账
- Backend：订阅 create/confirm 与 Apple IAP receipt/transaction 校验（新接口，entry 里记「待设计」）
- App Store Connect：订阅商品、隐私/商业模式 Review Notes、描述与 marketing URL 策略再审

### 明确不在本条「恢复」范围

- 手动开单 [`ManualOrderBillingStep`](xituan_app_merchant/src/screens/manual-order/ManualOrderBillingStep.tsx) 的「Apple Pay」是**记账方式枚举**，不是向顾客收 Apple Pay；与 PassKit 订阅结账无关
- Customer App 实物结账 Apple Pay 不在本 entry

## Registry 行（写入时）

| ID | 状态 | 简述 | 目标 | 下一动作 |
|----|------|------|------|----------|
| `merchant-ios-iap-pay-restore` | planned | Merchant iOS 上 IAP 时恢复订阅购买 UI/原生能力；勿再开 Stripe 订阅结账 | **mid** | 产品拍板 IAP 商品与后端校验后再开 |

## 与删号 / 过审回信

本条只记 **IAP 恢复笔记**。删号 7 天/匿名化、Resolution Center 四问仍按前序对话，不阻塞本台账。

## 落地动作（你确认本 plan 后）

1. 写 `xituan_agent/devGuide/planned-work/entries/2026-11-merchant-ios-iap-pay-restore.md`
2. 更新 `registry.md` Active 表 + Last updated
3. 不改 merchant App / 不挪 Cursor plan（除非另有删号 plan 定稿）
