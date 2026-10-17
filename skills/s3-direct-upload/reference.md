# S3 直传约定

Last updated: 2026-10-17

Skill checklist 在同目录 `SKILL.md` 与 `../s3-direct-upload-web|ios|android/SKILL.md`。主机和安全约束以本文为准。

## 决定

不新建 `upload[-env].lancety.com`。这个名字只出现在 2026-09 / 2026-10 的规划草稿里，没有 DNS、证书或 CloudFront，不算已落地的基础设施。

上传只做一套。文件类型不决定 PUT 主机，只决定 `complete` 之后的展示主机。客户端只 PUT 不透明的 `putUrl`，不按网页 / iOS / Android / 微信换 Host。

| 用途 | 主机 | 规则 |
|------|------|------|
| PUT（目录图、OpenIM 等新直传） | 现有 `content[-env]`；微信用 `wechatContent` | 只在暂存前缀的独立 behavior 上允许写入 |
| 展示：图片、视频、音频 | `media` / `wechatMedia` | 只读，禁止 PUT |
| 展示：其它文件 | `content` / `wechatContent` | 读 behavior 与写 behavior 分开 |

主机常量在 `xituan_codebase/constants/site-domain.ts`（各仓 submodule 同一份）。

## 为什么 media 不能拿来上传

`media` 是公开展示 CDN：默认 behavior 只有 `GET` / `HEAD` / `OPTIONS`，OAC 桶策略只有 `s3:GetObject`，缓存是 CachingOptimized。在这条 behavior 上打开 PUT，写入用的是 distribution 身份而不是上传者身份，对象一旦写入就能被公开 GET，并且和图片边缘缓存混在一起。

demo 的 content distribution 同样是只读 OAC，见 `xituan_agent/aws-setup/demo/demo-media-stack.yaml`。差别是 content 的定位是「直读 S3、不做图片变换」，已经承担非图片文件和 OTA 的阅读；写入行为加在它上面，而不是加在图片缓存上。

## 为什么不再加 upload 子域

单独子域可以把写 distribution 和读 distribution 拆开，但要新的 DNS、证书、CloudFront 别名，以及微信 request 合法域名。`content[-env].lancety.com` 已经在用。只要下面的写入约束全部成立，就不需要第三个主机。

这些约束缺任何一条，就不要在 content 上开 PUT，那时才重新考虑独立写域名：

1. 默认 behavior（`/*`）保持 `GET` / `HEAD` / `OPTIONS`，不加 PUT。
2. 只有暂存前缀（例如 `/_incoming/*`）有单独 behavior：方法含 PUT，缓存策略为 CachingDisabled。
3. 该 behavior 必须校验调用方（CloudFront 签名 URL 或等价的短时授权）。不能匿名 PUT。
4. 桶策略 `s3:PutObject` 只授予这个 distribution，且资源只到暂存前缀。
5. 禁止把 S3 SigV4 URL 的 Host 改成 `content` 或 `media`。签名绑 Host。content 上的 `putUrl` 由这条 behavior 自己签发，不是 `getSignedPutUrl` 改个域名。
6. 源对象写在暂存前缀。`complete` 成功后再落到展示 key。不要把未确认对象放在公开 CDN 的最终路径上。

这条写入 behavior 还没配。配好之前，目录图和 OpenIM 维持 multipart，不要开始直传。

## 为什么不按类型做两套上传

图片 / 视频 / 音频走 media、其它文件走 content，指的是展示 URL，不是两套 presign。再拆一套 PUT 只会多一份 behavior、签名和微信 request 域名，客户端仍然只认一个 `putUrl`。

当前 OpenIM 展示只把 `epOpenimMessageKind.IMAGE` 送到 media，其余（含视频、音频）送到 content，见 `openim-attachment-cdn.util.ts` 的 `resolveDomainKey`。视频和音频改到 media 展示时，只改这个展示选择，不新增上传实现。

## 协议

1. `presign` → `{ putUrl, sourceKey }`（外加该 API 的 bind 字段）。
2. 客户端 PUT 文件字节到 `putUrl`。不解析、不改 Host、不跟重定向换 Host。
3. `complete` 只返回 canonical（图片要等 IMAGE_NORMALIZE sync 成功）。失败不把源 key 交给表单。
4. 业务保存只提交这段文本，不再带文件。

编排放在 `xituan_codebase`。各端只提供 transport。页面里不要再写一套三步流程。

微信 PUT 用 `wx.request`，走 request 合法域名里的现成 `content*.lancety.com`，不用 `wx.uploadFile`，也不把 `amazonaws.com` 配进合法域名。

## 已经在跑的直传（不要顺手改主机）

| 通道 | 状态 | PUT |
|------|------|-----|
| CMS 支出批量 OCR | 已上线 | `s3UploadManager.getSignedPutUrl` → `s3-{region}.amazonaws.com` |
| App OTA CLI `ota:prepare` | 已上线（发布机，不是手机） | 同上；对象前缀 `app-ota/`，阅读走 content |
| 目录图 / OpenIM 附件 | 未做；OpenIM 预签名曾落地并已整段回退 | 按本文，等 content 写入 behavior |

支出和 OTA 继续用现有 S3 预签名。不要在同一次改动里把它们迁到 content behavior。

AWS SDK v3.729+ 会把 CRC32 塞进 `PutObject`。浏览器和 Expo 的 PUT 填不了 `x-amz-checksum-crc32=AAAAAA==`，表现是 `ERR_CONNECTION_RESET` 或 403。`s3-config.util.ts` 保持 `requestChecksumCalculation` / `responseChecksumValidation` 为 `WHEN_REQUIRED`；`getSignedPutUrl` 继续把 checksum 相关头放进 `unhoistableHeaders`。新的 content 签名同样不能把灵活校验和查询参数放进客户端 URL。PUT 的 `Content-Type` 必须和 presign 记录的一致。

## 端上传输（细节在对应 skill）

| 端 | 做法 |
|----|------|
| 网页 | `fetch` PUT，body 是 `File` / `Blob`，不是 FormData；不带 cookie。支出面板 `putFileToPresignedUrl` 对瞬时断连最多重试 3 次。跨域 PUT 需要目标主机 CORS 放行 `PUT` 和 `Content-Type`。 |
| iOS | Expo 57 `File.upload`：`httpMethod: 'PUT'`（默认是 POST）、`UploadType.BINARY_CONTENT`、`sessionType: 'foreground'`（默认 background，JS 被挂起后就叫不了 `complete`）。选图沿用 `preferredAssetRepresentationMode: Compatible`，HEIC 按 JPEG 字节和 `image/jpeg` 来签。`putUrl` 是 HTTPS，不加 ATS 例外。 |
| Android | 与 iOS 同一个 JS 调用。`content://` 先拷到缓存 `file://` 再 PUT。`sessionType` 只作用于 iOS，共享 helper 仍传 `foreground`。`usesCleartextTraffic` 只服务本地 API，上传不走明文。 |

顾客端和商家端共用这一份 Expo transport，不要各写一套状态机。

## 相关文档

- [目录图直传分层](../../../xituan_agent/devGuide/catalog-image-direct-upload-standard.md)
- [目录图台账](../../../xituan_agent/devGuide/planned-work/entries/2026-10-catalog-image-direct-upload.md)
- [OpenIM 附件台账](../../../xituan_agent/devGuide/planned-work/entries/2026-09-openim-attachment-direct-upload.md)
- [media / content / SIH 主机](../../../xituan_agent/devGuide/media-cdn-sih-domain-split.md)
