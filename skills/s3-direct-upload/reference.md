# S3 直传约定

Last updated: 2026-11-13

Skill checklist 在同目录 `SKILL.md` 与 `../s3-direct-upload-web|ios|android/SKILL.md`。主机和安全约束以本文为准。

## 决定

上传只做一套，走单独的写 CloudFront。文件类型不决定 PUT 主机，只决定 `complete` 之后的展示主机。客户端只 PUT 不透明的 `putUrl`。

| 用途 | 主机 | 规则 |
|------|------|------|
| PUT（各端） | `upload[-env].xituan.com.au`；prod 另有 `upload.lancety.com`（同一条分布的别名） | 短时 CloudFront 签名。对象 key 必须在 `upload-incoming/` |
| 展示：图片、视频、音频 | `media` / `wechatMedia` | 只读 |
| 展示：其它文件 | `content` / `wechatContent` | 只读 |

2026-11-09 已用 CLI 建好四条写分布。2026-11-13 **框架已落地**（签发 + 路由 + codebase 编排 + 各端 transport）；业务 purpose（目录图 / OpenIM）后置。未签名的 PUT 仍为 403。

| 环境 | 分布 | 别名 | 源站桶 |
|------|------|------|--------|
| dev | `E2LJPK638S3P2C` | `upload-dev.xituan.com.au` | `xituan-dev` |
| staging | `E3L7IFO3MDZL81` | `upload-staging.xituan.com.au` | `xituan-staging` |
| demo | `E3BQY4W5UZ9B6T` | `upload-demo.xituan.com.au` | `xituan-demo` |
| prod | `E2XHQ1JF0LAKQ7` | `upload.xituan.com.au`、`upload.lancety.com` | `xituan` |

证书复用已有的 `*.xituan.com.au` + `*.lancety.com`（us-east-1 `67d4bb81-8175-4e5d-ab42-7ead1714d2e5`）。OAC 复用读分布那条 `E2E06YZAM1GZWU`。缓存策略 CachingDisabled。源请求策略 `xituan-upload-s3-put`（`0b07d2fe-0dd8-4f8c-812d-571081d6b0e1`）只转发 `Content-Type` 和 CORS 预检头。签名密钥组 `6403e444-e55e-46b6-b670-983e28665ee9`，公钥 `K1MR7UEC8SHABK`。私钥在 Secrets Manager `xituan/cloudfront/upload-signing`（ap-southeast-2），**不要写入仓库或 `.env`**。Secret 名在代码常量 `CLOUDFRONT_UPLOAD_SIGNING_SECRET_ID`。

IAM：组 `s3_xituan` 内联策略 `CloudFrontUploadSigningSecretRead`（`GetSecretValue` / `DescribeSecret` 仅该 secret）。本地 `S3_KEY` 用户已可读取。

`xituan.com.au` 的四条别名在 Route53。`upload.lancety.com` 的 CNAME 在 lancety DNS（目标 `d2vz7boq1ttcgk.cloudfront.net`）。微信 request 合法域名已含 `upload.lancety.com`。

桶策略 Sid `AllowCloudFrontUploadOACPut`：只允许该写分布对 `upload-incoming/*` 做 `s3:PutObject`。没有授 `GetObject`。demo 同 Sid 在 `demo-media-stack.yaml`。

主机：`site-domain.upload` / `site-domain.wechatUpload`（prod 微信用 `https://upload.lancety.com`；非 prod 的 wechatUpload 与 upload 同为 xituan 主机）。

## 测试

| 类型 | 命令 |
|------|------|
| Jest unit + API integration（mock CF/S3） | `cd xituan_backend && npm test -- --testPathPattern=direct-upload` |
| 真链烟测（Secrets Manager + CloudFront PUT + S3 HEAD） | `npx ts-node --transpile-only scripts/probe-direct-upload.ts` |
| 孤儿 TTL（`upload-incoming/` 1 天过期） | `npx ts-node --transpile-only scripts/s3-upload-incoming-lifecycle-24h.ts` |

## complete（purpose=image）

1. HEAD `sourceKey`（须在 `upload-incoming/` 且归属当前 actor）  
2. **Copy** 到业务前缀（`merchant/…` 或 `user/…`，按 bindKind）  
3. IMAGE_NORMALIZE：`waitNormalize=true` → sync 等终态回 **canonical**；`false` → async，回 `jobId` + `promotedKey`，`path` 为空  
4. 默认 wait：头像 / OpenIM / 打印模板 = true；目录图/logo 等 = false（客户端可覆盖）  
5. **no-wait 展示标准**：落库 `path \|\| promotedKey`；缩略图先 `_w*`，**`onError` → 原图 URL**（promote 后原对象可读）。详见 `SKILL.md`；Site 购物车备注图已落地

## 框架模块（2026-11-13）

| 层 | 路径 |
|----|------|
| 契约 | `xituan_codebase/constants/direct-upload.enum.ts`、`typing_api/direct-upload.type.ts` |
| 编排 | `xituan_codebase/utils/direct-upload.util.ts` → `uploadOne` / `uploadMany` |
| 签发 | `xituan_backend/src/shared/utils/cloudfront-upload-signing.util.ts` |
| 服务 | `xituan_backend/src/domains/direct-upload/` |
| 商户路由 | `POST /api/admin/direct-upload/presign`、`.../complete` |
| 顾客路由 | `POST /api/direct-upload/presign`、`.../complete`（**按路由鉴权**，禁止在挂 `/api` 的 router 上 `router.use(authenticate)`） |
| Web transport | CMS `src/utils/direct-upload-web.transport.ts`；Site 同名（Site 的 `NEXT_PUBLIC_API_URL` 已含 `/api`，path 勿再加 `/api`；Authorization 用 `authStoreManager`） |
| RN transport | 商家/顾客 App `src/utils/direct-upload-rn.transport.ts` |
| 微信 transport | `packageMerchant/utils/direct-upload.wechat.transport.ts`；顾客 `utils/direct-upload-customer.wechat.transport.ts` |
| CLI 探测 | `xituan_backend/scripts/probe-direct-upload.ts`（`NODE_ENV=development npx ts-node --transpile-only scripts/probe-direct-upload.ts`） |

框架期 `purpose=probe`：须登录；≤256KB；MIME 白名单。`complete` 只 HEAD，返回 `path = sourceKey`。新业务加 `epDirectUploadPurpose` 值 + complete hook，不要新开签名栈。

## 写分布和读分布分开

`media` 和 `content` 是公开读 CDN（CachingOptimized，OAC 只有 `s3:GetObject`）。upload 是另一条分布：允许 PUT，不缓存，必须带 CloudFront 签名，桶策略只给 `upload-incoming/*`。

不要把 `upload` 别名加进读分布，也不要 CNAME 到 `xituan.com.au` 的 API。签名绑的是 upload 主机。API 中介不会把 PUT 转给 S3。

SIH 上手工挂过的 `images` 别名（prod 的 `images.*` 仍在 SIH，dev/staging/demo 的 `images-*` 已是读分布别名）与这条写分布无关，不要照搬。

`putUrl` 由后端用 Secrets Manager 里的私钥签到 `https://upload…/upload-incoming/…`。不要用 `getSignedPutUrl` 再改 Host。支出 OCR 和 OTA CLI 仍走 `amazonaws.com`，不在这次迁。

## 为什么不按类型做两套上传

图片 / 视频 / 音频走 media、其它文件走 content，指的是展示 URL，不是两套 presign。再拆一套 PUT 只会多一份 behavior、签名和微信 request 域名，客户端仍然只认一个 `putUrl`。

当前 OpenIM 展示只把 `epOpenimMessageKind.IMAGE` 送到 media，其余（含视频、音频）送到 content，见 `openim-attachment-cdn.util.ts` 的 `resolveDomainKey`。视频和音频改到 media 展示时，只改这个展示选择，不新增上传实现。

## 协议

1. `presign` → `{ putUrl, sourceKey }`（外加该 API 的 bind 字段）。
2. 客户端 PUT 文件字节到 `putUrl`。不解析、不改 Host、不跟重定向换 Host。
3. `complete` 只返回 canonical（图片要等 IMAGE_NORMALIZE sync 成功）。失败不把源 key 交给表单。
4. 业务保存只提交这段文本，不再带文件。

编排放在 `xituan_codebase`。各端只提供 transport。页面里不要再写一套三步流程。

微信 PUT 用 `wx.request` 打 `upload.lancety.com`（request 合法域名），不用 `wx.uploadFile`，也不把 `amazonaws.com` 或 `content` / `media` 配进 request 合法域名。

## 已经在跑的直传（不要顺手改主机）

| 通道 | 状态 | PUT |
|------|------|-----|
| CMS 支出批量 OCR | 已上线 | `s3UploadManager.getSignedPutUrl` → `s3-{region}.amazonaws.com` |
| App OTA CLI `ota:prepare` | 已上线（发布机，不是手机） | 同上；对象前缀 `app-ota/`，阅读走 content |
| 目录图 / OpenIM 附件 | 业务未接；写 CloudFront 已建 | `putUrl` 签到 `upload` 主机的 `upload-incoming/` |

支出和 OTA 继续用现有 S3 预签名。不要在同一次改动里把它们迁到 upload 分布。

AWS SDK v3.729+ 会把 CRC32 塞进 `PutObject`。浏览器和 Expo 的 PUT 填不了 `x-amz-checksum-crc32=AAAAAA==`，表现是 `ERR_CONNECTION_RESET` 或 403。`s3-config.util.ts` 保持 `requestChecksumCalculation` / `responseChecksumValidation` 为 `WHEN_REQUIRED`；`getSignedPutUrl` 继续把 checksum 相关头放进 `unhoistableHeaders`。upload 的 `putUrl` 是 CloudFront 签名，不是 S3 SigV4，不要带 `x-amz-checksum-*`。S3 预签名路径仍不要把灵活校验和放进客户端 URL。PUT 的 `Content-Type` 必须和 presign 记录的一致。

## 端上传输（细节在对应 skill）

| 端 | 做法 |
|----|------|
| 网页 | `fetch` PUT，body 是 `File` / `Blob`，不是 FormData；不带 cookie。支出面板 `putFileToPresignedUrl` 对瞬时断连最多重试 3 次。跨域 PUT 需要目标主机 CORS 放行 `PUT` 和 `Content-Type`。 |
| iOS | Expo 57 `File.upload`：`httpMethod: 'PUT'`、`BINARY_CONTENT`、`sessionType: 'foreground'`。**2026-10：** 勿用 RN `fetch`+FormData 传文件；相册权限 `PHOTOS_PERMISSION`；HEIC 用 Compatible / 真实 JPEG 字节。细节见 `s3-direct-upload-ios`。 |
| Android | 与 iOS 同一 JS 调用；`content://` → 缓存 `file://`。**2026-10：** 产品多图选图勿用 `expo-image-picker`（ActivityResultLauncher 崩溃），用 `DocumentPicker`（见 `merchant-product-image-pick.util.ts`）。细节见 `s3-direct-upload-android`。 |

顾客端和商家端共用这一份 Expo transport，不要各写一套状态机。

## 相关文档

- [S3 桶访问 / 私有前缀](../../../xituan_agent/devGuide/s3/README.md) — CDN 匿名 GET Deny；改桶策略必看
- [目录图直传分层](../../../xituan_agent/devGuide/catalog-image-direct-upload-standard.md)
- [目录图台账](../../../xituan_agent/devGuide/planned-work/entries/2026-10-catalog-image-direct-upload.md)
- [OpenIM 附件台账](../../../xituan_agent/devGuide/planned-work/entries/2026-09-openim-attachment-direct-upload.md)
- [media / content / SIH 主机](../../../xituan_agent/devGuide/media-cdn-sih-domain-split.md)
