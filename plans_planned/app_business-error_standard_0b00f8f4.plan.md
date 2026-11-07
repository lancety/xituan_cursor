---
name: App business-error standard
overview: "【planned / high — WeChat C】业务错误统一 code→i18n。Customer/Merchant App 与 CMS 已有 handler util。下一优先：微信 C 端审计并去掉 toast(err.message)。微信「我的网店」商户管理后续删除，商户面板 handler 不投入；退款等共享路径仍跟 C 端/CMS。"
todos:
  - id: wechat-c-handler-audit
    content: "【high】微信 C 端：审计 SimpleErrorHandler；业务 API 禁止 toast(err.message)，改 code→business-error-message"
    status: pending
  - id: wechat-c-i18n-gaps
    content: 微信 C 端补齐缺失 businessError 文案（含退款超额等）
    status: pending
  - id: clarify-server-contract
    content: 文档明确：服务端契约是 code+payload/lineErrors；message 仅旧端兼容
    status: pending
  - id: backend-refund-service
    content: payment-business.service processRefund 超限改抛 BusinessError（若尚未）
    status: pending
  - id: cms-scenario-migrate
    content: CMS：按场景扫漏网 err.message（已有 cmsBusinessErrorMessageUtil）
    status: pending
  - id: merchant-scenario-migrate
    content: Merchant App：扫漏网 catch（已有 appBusinessErrorHandleUtil）
    status: pending
  - id: wechat-merchant-handler
    content: "【取消投入】微信「我的网店」商户面板将删除；不做专项 handler，除非共享 C 端路径需要"
    status: cancelled
  - id: customer-app-util
    content: Customer App businessErrorMessageUtil
    status: completed
  - id: merchant-handle-util
    content: Merchant App appBusinessErrorHandleUtil
    status: completed
  - id: cms-handler-util
    content: CMS cmsBusinessErrorMessageUtil
    status: completed
isProject: false
---

# 业务错误文案统一

**2026-11-07**

| 端 | 状态 |
|----|------|
| Customer App | 已有 `businessErrorMessageUtil` |
| Merchant App | 已有 `appBusinessErrorHandleUtil` |
| CMS | 已有 `cmsBusinessErrorMessageUtil` |
| **微信 C 端** | **planned-work high**：审计并改掉 `toast(err.message)` |
| 微信「我的网店」商户面板 | **将删除** → 不专项投入 |

## 原则

- API 只回 `code` + 结构化 payload / `lineErrors`；客户端 i18n。
- 不向用户展示后端 `message` 原文。
