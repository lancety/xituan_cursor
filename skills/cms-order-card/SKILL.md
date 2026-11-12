---
name: cms-order-card
description: >-
  CMS and merchant-app order list cards (CmsActivityOrderCard / ActivityOrderCard):
  avatar/nickname, status background tone, order/payment badges (btn vs state),
  pack-ready, fulfillment, payment ref, products. Use when changing 订单卡片,
  order card modal/detail header, payment badge clickability (committed vs pending),
  or aligning CMS ↔ merchant card logic.
---

# 订单卡片（CMS + Merchant App）

跨端业务逻辑以本 skill 为准；视觉壳可平台差异（拨号、地图、商品缩略图等）。

| 端 | 列表卡权威组件 |
|----|----------------|
| CMS | `CmsActivityOrderCard` |
| Merchant Expo | `ActivityOrderCard` |

CMS 弹窗 summary（`OrderEditModal`）必须与 **CMS 列表卡**同源。Merchant 详情页复用同一 `ActivityOrderCard`。

Badge 颜色体系见 **`badge-style-conventions`**。

---

## 文件地图

### CMS

| 角色 | 路径 |
|------|------|
| 卡片组件 | `xituan_cms/src/components/orders/CmsActivityOrderCard.tsx` |
| 卡片样式 | `…/CmsActivityOrderCard.module.css`（`data-status-tone` 背景） |
| Props | `…/cms-activity-order-card.type.ts` |
| 列表视图拼装 | `xituan_cms/src/utils/cmsOrderCardView.util.ts`（`buildCardView` / `resolveNickname`） |
| Badge 可点与 class | `xituan_cms/src/utils/activityOrderBadgeAction.util.ts` |
| 状态 → 卡片底色 tone | `xituan_cms/src/utils/activityOrderDetailModal.util.ts`（`orderStatusTone`） |
| 履约行文案 | `xituan_cms/src/utils/cms-order-card-fulfillment-display.util.ts` |
| 列表挂载 | `xituan_cms/src/pages/orders.tsx`、`…/orders/manual.tsx` |
| 弹窗（须对齐卡片） | `…/OrderEditModal.tsx` + `OrderDetailModal.module.css` |
| 活动入口委托弹窗 | `…/ActivityOrderDetailModal.tsx`（传 `listUser`） |
| 卡片动作宿主 | `…/CmsOrderCardActionsHost.tsx` |

活动展开面板另有内联卡（`ActivityOrdersExpandPanel`）；改权威逻辑时优先列表卡 + 上述 util，再让面板/弹窗跟上。

### Merchant App

| 角色 | 路径 |
|------|------|
| 卡片组件 | `xituan_app_merchant/src/components/ActivityOrderCard.tsx` |
| 视图拼装 | `…/utils/merchant-activity-order-card-view.util.ts` |
| Badge presentation | `…/utils/merchant-order-badge.util.ts` |
| Badge 动作 | `…/utils/merchant-order-actions.util.ts` |
| 文案 /（待废弃）旧底色 | `…/utils/merchant-order-labels.util.ts` |
| 商品行 | `…/utils/merchant-activity-order-product-line.util.ts` |
| 动作宿主 | `…/components/OrderActionsHost.tsx` |
| i18n 键 | `orders.status.*` / `orders.payment.*`（`src/i18n/messages/{en,zh_cn,zh_tw}.json`） |

---

## 布局

- **空白区点击** → 打开详情（badge / 金额 / footer 不冒泡）
- **≥660**：左信息 + 右商品列（共享 `order-card-layout.enum`）
- **排号** `seqNumber`：活动主题 seq badge

---

## 元素：设计与逻辑

### 1. 头像 + 昵称

| 项 | 规则 |
|----|------|
| 昵称优先级 | `user.nickname` → `billingPartyTitle` → `deliveryAddressSnapshot.contactName` → `orderNumber` |
| 头像 | `user.avatarUrl`；无则 `nameInitial`（昵称首字） |
| CMS 列表 | `userMap` → `buildCardView`；弹窗传 `listUser`（可 batch 兜底） |
| Merchant | 行上常带 `order.user`；fallback 同上优先级 |

收货人姓名只在履约区，不顶替头昵称。

### 2. 卡片背景色（订单状态）

- Tone：**仅用** `orderStatusTone` 语义：`warning` / `neutral` / `success` / `muted` / `error`
- **禁止**旧 `cardStatusTone`（pending/flow/delivered/closed/risk）作为列表卡/详情卡底色

| 状态族 | tone |
|--------|------|
| PENDING / READY_FOR_DELIVERY | warning |
| PROCESSING / SHIPPED / IN_DELIVERY | neutral |
| DELIVERED | success |
| CANCELLED / EXPIRED / DELETED | muted |
| REFUNDED | error |

CMS：`CmsActivityOrderCard.module.css` + 弹窗 `.detailSummaryCard` 同色。  
Merchant：`ActivityOrderCard` `resolveCardBg` 应按**同一套 tone 名**上色（勿再走 pending/flow 键）。

### 3. 订单状态 badge

- 可点：PENDING / PROCESSING / READY_FOR_DELIVERY / SHIPPED / IN_DELIVERY / EXPIRED
- 可点 → **badge 按钮**；不可点 → **badge 状态**
- **不可点不绑定**点击（`undefined` / 不包可点 Pressable）

### 4. 支付状态 badge（最终方案）

Checkout：cash / bank transfer → `paymentStatus = committed`（承诺支付），不是 `pending`。

| `paymentStatus` | 样式 | 可点 | 动作 |
|-----------------|------|------|------|
| **`committed`** | badge 按钮（amber） | 是 | `confirm` |
| **`pending`** | badge 状态 | **否** | 无 |
| `success` / `partially_refunded` | badge 按钮 | 是 | `refund` |
| 其余 | badge 状态 | 否 | 无 |

- `isPaymentCommittedConfirmActionable` ≡ `paymentStatus === committed`
- `resolvePaymentBadgeAction` 判定顺序：先 **`confirm`（committed）**，再 **`refund`**（状态互斥，与文档表一致；勿把 pending 当 confirm）
- 不可点不绑事件

### 5. 支付方式图标 + 转账参考号

- 仅 bank transfer 展示参考号；点击复制
- CMS：`cmsCopyPaymentReferenceUtil.resolveDisplayRef`

### 6. 金额

- 展示格式化 AUD（可带调整标记）
- **点击 → 支付流水**（CMS：`onAmountClick` → 支付历史；Merchant：`onAmountPress` → `OrderActionsHost` 支付流水 sheet）
- 无流水回调时金额**不可点**（不要 fallback 进详情）

### 7. 下单时间 + 备妥（pack-ready）

- 下单时间：列表卡 `MM/DD HH:mm`（i18n 包裹）
- **备妥**：头区在下单时间旁展示（` · ` 连接）；Merchant PREORDER 用确认配送日，否则 `estimatedPackReadyAt`；CMS 列表用 `estimatedPackReadyAt` + `packReadyWithDate` i18n

### 8. 履约

- 自提 / 配送联系人与地址；「更改」受权限与 `canChangeFulfillment` 控制
- Merchant 可额外拨号、外开地图、详情内嵌地图（平台能力，不强迫 CMS）

### 9. 商品行 + 备注

- 多语言名 / 自定义名 / 数量（Merchant 可另有图、规格、archived）
- `order.note`

### 10. 状态文案 i18n

- **两端都必须走 locale**；卡片 badge 用 `t('orders.status.*')` / `t('orders.payment.*')`（CMS：`order.paymentStatus.*`）。
- `payment.pending` / `paymentStatus.pending` 文案定为 **未支付**（en: Unpaid），不要用「待支付」。

---

## 弹窗 / 详情对齐清单

1. 昵称 = resolveNickname 同源优先级  
2. 头像 = 用户 avatar  
3. 底色 = `orderStatusTone`  
4. 支付 badge = committed 可点 / pending 不可点  
5. 不可点不绑事件  
6. 备妥展示与列表卡一致（CMS 补齐）

---

## Agent 改动清单

1. 先读本 skill；勿恢复 pending→confirm。  
2. 改逻辑优先改 badge util / card view util，再改组件。  
3. CMS ↔ Merchant 业务规则同步；改一端时检查另一端。  
4. Badge 用 `badgeStyleUtil` / `BadgeBtn`·`BadgeState`。  
5. 卡片状态文案必须走 i18n。  
6. CMS 列表卡补备妥，看齐 Merchant。  
