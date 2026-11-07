---
name: Last Used Delivery Address
overview: "【planned / low】移除「默认地址」，配送下单成功写 lastUsedDeliveryAt；结账与地址列表用最近配送地址预填。自提不更新。三端 Site/WeChat/Customer App。优先级低，不阻塞其它需求。"
todos:
  - id: codebase-types
    content: "xituan_codebase: iUserAddress 改 Omit+lastUsedDeliveryAt，删 customers.type setDefaultAddress，新增 userAddressUtil"
    status: pending
  - id: db-migration-phase1
    content: "Phase1 迁移：仅 ADD last_used_delivery_at + 从 is_default 回填；保留 is_default 列"
    status: pending
  - id: backend-api-phase1
    content: "Phase1 backend：markLastUsed 双写 is_default；GET 仍返回 isDefault+lastUsedDeliveryAt；保留 set-default"
    status: pending
  - id: db-migration-phase3
    content: "Phase3（WeChat 审核上线后）：DROP is_default；删 set-default API"
    status: pending
  - id: backend-api-phase3
    content: Phase3 backend：entity 移除 isDefault，停止双写
    status: pending
  - id: order-hook
    content: createSingleOrder 配送成功后调用 markLastUsedDeliveryAddress
    status: pending
  - id: wechat-ui
    content: WeChat 地址管理/selector/commerce：移除设默认，预填与 badge 改用 lastUsedDeliveryAt
    status: pending
  - id: site-ui
    content: Site addresses/checkout/address.service：同上，保留门店 isDefault 自提逻辑
    status: pending
  - id: customer-app-ui
    content: Customer App 地址管理/结账预填对齐 lastUsedDeliveryAt
    status: pending
  - id: sync-submodules
    content: 按 multirepo 流程同步相关仓 xituan_codebase submodule
    status: pending
isProject: false
---

# 用户地址：默认地址 → 最后使用配送地址

**状态（2026-11-07）：** planned-work **low**；未开工。post-deploy 条目 `user-address-last-used-delivery` 仍适用（Phase1 双写 → Gate 后 DROP `is_default`）。

## 规则

- 配送订单成功创建时写 `lastUsedDeliveryAt`。
- 结账 / 地址列表用「最近配送地址」预填与展示。
- **自提下单不更新**该字段。
- 覆盖 Site / WeChat / Customer App。

## 实施备忘

详见历史方案正文（Phase1 ADD + 双写；WeChat 全量后再 Phase3 DROP）。主动启动前先读 `xituan_agent/devGuide/post-deploy-ledger/entries/2026-06-user-address-last-used-delivery.md`。
