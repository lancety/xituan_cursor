---
name: Direct upload business migration
overview: 【planned / high】框架已完成。顺序：complete+normalize(等终态) → TTL 24h → 按端迁 微信→Site→顾客App→CMS→商家App → 打印+IM → post-deploy。我的网店忽略；OCR 搁置；先 planned。
todos:
  - id: purpose-hooks
    content: purpose=image + bindKind + complete 双模式（promote + IMAGE_NORMALIZE）
    status: completed
  - id: orphan-ttl
    content: upload-incoming 孤儿 TTL 24h（lifecycle 脚本；待对各桶执行）
    status: completed
  - id: per-client-merge-note-avatar
    content: 备注图/头像不单独立项 — 随该端当前目录图/顾客面迁移一并改
    status: in_progress
  - id: expo-cms-catalog
    content: 按端迁：微信顾客 done → Site done → 顾客App done → CMS done → 商家App done
    status: completed
  - id: wechat-customer-note-avatar
    content: 微信顾客 cart_note_images + user_avatar 直传（waitNormalize）
    status: completed
  - id: site-customer-note-avatar
    content: Site cart_note_images（no-wait + 购物车原图回退）+ user_avatar（wait）直传
    status: completed
  - id: customer-app-note-avatar
    content: 顾客 App cart_note_images（no-wait + 原图回退）+ user_avatar（wait）直传
    status: completed
  - id: print-temp-and-openim
    content: 打印模板 CMS done；OpenIM CMS+Site done（微信/App 仍 multipart）
    status: in_progress
  - id: wechat-merchant-storefront-skip
    content: 微信商家「我的网店」深链下全部功能 — 忽略直传迁移（不排期）
    status: cancelled
  - id: skill-cms-form
    content: skill cms-form-image-upload 改为直传 checklist
    status: completed
  - id: post-deploy-multipart
    content: 各端发版无流量后另开 post-deploy 下线旧 multipart 路由
    status: pending
isProject: false
---

# 直传业务迁移清单（框架已完成）

## 已完成（勿再开新框架工作）

| 项 | 位置 |
|----|------|
| CloudFront 写分布 + IAM + site-domain upload/wechatUpload | [s3-direct-upload/reference.md](../skills/s3-direct-upload/reference.md) |
| Backend signer + `DirectUploadService` + admin/customer 路由（`purpose=probe`） | `xituan_backend/src/domains/direct-upload/` |
| codebase `directUploadUtil` + 各端 Transport | CMS/Site/商家App/顾客App/微信 |
| Jest + probe 烟测 | `tests/**/direct-upload*`；`scripts/probe-direct-upload.ts` |
| 框架 plan / 测试 plan | [plans_completed](../plans_completed/) |

权威约定与平台坑：skill `s3-direct-upload` / `s3-direct-upload-web|ios|android`。

总表：[image-upload-surface-inventory.md](../../xituan_agent/devGuide/image-upload-surface-inventory.md)

## 本 plan 范围

**做**：purpose/complete hook + 薄 UI；对齐台账。

**不做 / 搁置**：

- 再建第二套 PUT 状态机
- **微信商家「我的网店」深链下的全部功能** — **忽略直传迁移**
- OCR 决定搁置（维持 SigV4）
- OTA；`media`/`content` 开写；历史 OpenIM Mongo `.png` 回填

## 已确认实施顺序（2026-11-13 修订）

1. purpose / bindKind + complete hook（IMAGE_NORMALIZE；**prod = Lambda**；**双模式 wait / no-wait**，见下）  
2. **孤儿 TTL = 24h**（`upload-incoming` 未 complete）  
3. **一次只做一端**：微信（顾客）→ Site → 顾客 App → CMS → 商家 App  
4. **打印模板** + **OpenIM** 同批后置  
5. post-deploy 拆旧 multipart  

### complete 双模式（已定方向）

| 模式 | 何时 |
|------|------|
| **wait** | 多处只认 CDN、对端要 URL、或本地句柄保不住（头像、OpenIM、打印模板、微信备注默认） |
| **no-wait** | CMS 目录图/logo、商家 App 目录图；Site 备注（购物车 `_w*` onError→原图） |

重判表见 [image-upload-surface-inventory.md](../../xituan_agent/devGuide/image-upload-surface-inventory.md) § complete 双模式。

## 清单分桶

| 桶 | 内容 | 何时 |
|----|------|------|
| complete + normalize | 双模式 wait/no-wait；禁止源 path 最终落库 | 步 1 |
| 孤儿 TTL | 24h | 步 2 |
| 按端迁移 | 微信顾客 **done** → Site **done** → 顾客App **done** → CMS → 商家App | 步 3（拆开做） |
| 备注图 / 头像 | 微信/Site **done**；其余并入各端 | 步 3 |
| ~~我的网店深链~~ | — | **忽略** |
| D+B | 打印模板 **done**；OpenIM **CMS+Site done**（微信/App 待迁） | 步 4 |
| OCR | — | 搁置 |

台账：

- 目录图：[catalog-image-direct-upload](../../xituan_agent/devGuide/planned-work/entries/2026-10-catalog-image-direct-upload.md)  
- OpenIM：[openim-attachment-direct-upload](../../xituan_agent/devGuide/planned-work/entries/2026-09-openim-attachment-direct-upload.md)  
- 标准：[catalog-image-direct-upload-standard.md](../../xituan_agent/devGuide/catalog-image-direct-upload-standard.md)

## 触发

主动启动时实施。微信顾客 / Site / 顾客 App / **CMS** / 商家 App 目录面已迁直传（2026-11-13）；**CMS 打印模板图**已直传（wait）；**OpenIM CMS+Site** 已直传（域内 presign/complete）；下一步微信/App OpenIM，再 post-deploy 下线 multipart。
