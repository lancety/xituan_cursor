---
name: Direct upload business migration
overview: 【planned / high】框架已完成。微信商家「我的网店」深链忽略；备注图/头像随各端迁移合并；CMS/Expo 目录图可做；打印模板+IM 同批后置；OCR 搁置；先 planned 不开工。
todos:
  - id: purpose-hooks
    content: purpose/bindKind + complete 挂 IMAGE_NORMALIZE sync（目录图/备注/头像共用）
    status: pending
  - id: per-client-merge-note-avatar
    content: 备注图/头像不单独立项 — 随该端当前目录图/顾客面迁移一并改
    status: pending
  - id: expo-cms-catalog
    content: Expo 商家 + CMS 目录图薄 Upload（Offer/Preorder → Product/News…）
    status: pending
  - id: print-temp-and-openim
    content: 打印模板 + OpenIM 附件同批后置
    status: pending
  - id: wechat-merchant-storefront-skip
    content: 微信商家「我的网店」深链下全部功能 — 忽略直传迁移（不排期）
    status: cancelled
  - id: orphan-ttl
    content: upload-incoming 未 complete 对象 TTL/孤儿清理
    status: pending
  - id: skill-cms-form
    content: skill cms-form-image-upload 改为直传 checklist
    status: pending
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

1. purpose / bindKind + complete hook（IMAGE_NORMALIZE sync）  
2. **按端迁移目录图等业务上传**（Expo 商家、CMS、Site/顾客 App、微信**顾客**包…）  
   - **备注图 / 头像**：不单独立项、不抢先；**随该端当前波次一并改掉**（该端有则合并进同 PR/同里程碑）  
3. **打印模板** + **OpenIM** 同批后置  
4. 孤儿 TTL  
5. post-deploy 拆旧 multipart  

**明确忽略**：微信商家「我的网店」深链全部功能。

## 清单分桶

| 桶 | 内容 | 何时 |
|----|------|------|
| A 目录图等（按端） | CMS、Expo 商家；微信顾客包若有非「我的网店」上传面 | 步 2 |
| 备注图 / 头像 | `cart_note_images` / `user_avatar` | **并入各端步 2**，不单独排期 |
| ~~微信商家「我的网店」深链~~ | 深链可达全部功能 | **忽略** |
| D+B | 打印模板 + OpenIM | **同批后置**（步 3） |
| OCR | 支出收据 | 搁置 |

台账：

- 目录图：[catalog-image-direct-upload](../../xituan_agent/devGuide/planned-work/entries/2026-10-catalog-image-direct-upload.md)  
- OpenIM：[openim-attachment-direct-upload](../../xituan_agent/devGuide/planned-work/entries/2026-09-openim-attachment-direct-upload.md)  
- 标准：[catalog-image-direct-upload-standard.md](../../xituan_agent/devGuide/catalog-image-direct-upload-standard.md)

## 触发

主动启动时实施；**先 planned，现在不改业务页**。
