---
name: merchant-subdomain-storefront
overview: "【completed 2026-11-07】子域店 URL 已交付；外域见活跃 site_host_surface Phase4。"
todos:
  - id: dns-vercel
    content: 配置 Vercel wildcard domains，并在 Route53 将 m/m-demo 子域委派给 Vercel nameservers。
    status: pending
  - id: codebase-domain
    content: 在 shared siteDomain/工具层增加 merchant storefront base host。
    status: pending
  - id: merchant-code-slug
    content: 将新商户申请的自动 APP code 改为用户输入的 storefront slug，并统一校验规则。
    status: pending
  - id: backend-cors-api
    content: Backend 放行 storefront wildcard origin，并新增按 merchant code 解析公开接口。
    status: pending
  - id: site-host-rewrite
    content: Site middleware 解析 Host 并 rewrite 到内部 store code 路由。
    status: pending
  - id: site-store-pages
    content: 新增 store/[merchantCode] 页面并复用现有 merchantId 页面组件。
    status: pending
  - id: auth-check
    content: 登录统一跳主域，暂不支持子域微信登录 callback wildcard。
    status: pending
isProject: false
---

# 商户子域商铺地址改造计划

## 目标形态
- 生产：`https://fresh-market.m.xituan.com.au` 打开商户 `code = fresh-market` 的商铺首页。
- Demo：`https://fresh-market.m-demo.xituan.com.au` 打开同一套商户 code 路由。
- 本地 Dev：只采用 hosts/DNS 子域方式测试，例如 `http://fresh-market.m-dev.xituan.com.au:3005`。不把 `localhost` path fallback 作为目标入口。

## 外部配置
- Vercel site project 添加 wildcard domains：`*.m.xituan.com.au`、`*.m-demo.xituan.com.au`。
- 按 Vercel 官方要求，wildcard domain 需要 Nameservers method 以便自动签发/续期 wildcard SSL 证书。
- 不迁移整个 `xituan.com.au`；在 Route53 的 `xituan.com.au` hosted zone 里只添加子域 NS 委派：
  - `m.xituan.com.au NS ns1.vercel-dns.com / ns2.vercel-dns.com`
  - `m-demo.xituan.com.au NS ns1.vercel-dns.com / ns2.vercel-dns.com`
- 委派后，`*.m.xituan.com.au` 和 `*.m-demo.xituan.com.au` 的解析与证书由 Vercel 管理；主域、backend、cms 等仍由 Route53 管理。
- Dev 不建议 Route53 指向 Vercel；本地测试通过 hosts 或本机 DNS 工具把测试子域指到本机。

## Shared Codebase 配置
- 在 [xituan_backend/submodules/xituan_codebase/constants/site-domain.ts](xituan_backend/submodules/xituan_codebase/constants/site-domain.ts) 增加 merchant storefront base host 字段，例如：
  - prod：`https://m.xituan.com.au`
  - demo：`https://m-demo.xituan.com.au`
  - dev：`http://m-dev.xituan.com.au:3005`，生成具体商铺地址时使用 `{code}.m-dev.xituan.com.au:3005`
- 如需生成链接，新增/复用 util 用 `merchant.code + baseHost` 拼出商铺地址，避免业务代码硬编码域名。

## 商户 Code / 子域 Slug 改动
- 将 `merchant.code` 明确作为 storefront slug 使用，不再让新商户申请默认生成 `APP-...`。
- 规则按用户输入的有意义子域值收紧：仅允许小写 `a-z`、数字 `0-9` 和短横线 `-`，长度 `3-32`，不允许开头/结尾是 `-`，不允许连续 `--`。
- 在 [xituan_backend/src/domains/merchant/services/merchant.service.ts](xituan_backend/src/domains/merchant/services/merchant.service.ts) 的 `createApplication()` 中使用请求里的 `code`，先 normalize 为 lowercase，再通过 `getByCode()` 做唯一性检查。
- 在 [xituan_backend/src/domains/merchant/controllers/merchant.controller.ts](xituan_backend/src/domains/merchant/controllers/merchant.controller.ts) 的申请提交校验里要求 `code` 必填，校验失败返回明确错误。
- 在 [xituan_cms/src/pages/apply-merchant.tsx](xituan_cms/src/pages/apply-merchant.tsx) 的“创建新商户申请”表单新增“商铺地址/子域名”输入框，展示预览：`{code}.m.xituan.com.au`；加入已有商户的 `merchantCode` 表单保持原语义。
- 在 [xituan_cms/src/lib/api/merchant-application.api.ts](xituan_cms/src/lib/api/merchant-application.api.ts) 的 request type 加 `code`，提交给 backend。
- 历史 `APP-...` 商户不强行迁移；它们不会自动获得漂亮子域。需要平台后续手动改 code 或单独做一次数据修正。

## Backend 改动
- 修改 [xituan_backend/src/app.ts](xituan_backend/src/app.ts) CORS origin 判断，允许环境对应 wildcard origin：
  - prod：`https://*.m.xituan.com.au`
  - demo：`https://*.m-demo.xituan.com.au`
  - dev 可选：`http://*.m-dev.xituan.com.au:3005`
- 在 [xituan_backend/src/domains/merchant/routes/public-merchant.routes.ts](xituan_backend/src/domains/merchant/routes/public-merchant.routes.ts) 增加公开接口，例如 `GET /api/public/merchants/by-code/:code`。
- 接口用现有 [xituan_backend/src/domains/merchant/services/merchant.service.ts](xituan_backend/src/domains/merchant/services/merchant.service.ts) 的 `getByCode()`，只返回 active 且 site checkout capable 的商户，返回 `merchantId/code/shopName/logo` 等前端必要字段。
- 登录策略：子域商铺内点击登录时跳转到主域 `xituan.com.au` 登录；本期不支持在 `*.m.xituan.com.au` 内完成微信网页登录 callback，因此不改 [xituan_backend/src/domains/user/controllers/auth.controller.ts](xituan_backend/src/domains/user/controllers/auth.controller.ts) 的 callback wildcard 白名单。

## Site 改动
- 修改 [xituan_site/src/middleware.ts](xituan_site/src/middleware.ts)：
  - 保留现有 locale redirect 行为。
  - 解析 `Host`，识别 `test123.m.xituan.com.au` / `test123.m-demo.xituan.com.au`。
  - 将子域请求 rewrite 到内部 route：`/{locale}/store/{merchantCode}`，地址栏仍保持子域。
- 新增 `xituan_site/src/app/[locale]/store/[merchantCode]/...` 路由组：
  - 首页、products、product detail、offers、preorder、news 等按现有 merchant 页面范围逐步覆盖。
  - 页面先调用 backend `by-code` 解析 `merchantCode -> merchantId`，再复用现有 `merchant/[merchantId]` 下的 client component。
- 保留现有 `/{locale}/merchant/{merchantId}`，用于兼容已有链接。
- 本地测试使用 hosts/DNS 子域入口，例如 `http://fresh-market.m-dev.xituan.com.au:3005`；middleware 需要能识别带端口的 `Host`。

## 可选 CMS/平台改动
- 如果 CMS 或平台需要展示“商铺地址”，在商户详情/列表中使用 shared util 生成：prod/demo 显示子域 URL，本地显示 fallback URL。
- 平台侧如果已有商户编辑 code 的入口，需要加同样的 slug 校验；如果没有入口，本次只覆盖新申请提交。

## 风险点
- `merchant.code` 将进入 URL 和 DNS label，必须避免大写、下划线、空格和特殊字符；`test123`、`test-shop` 合法，`Test_123` 不合法。
- 登录态先不做跨子域共享；子域点击登录跳主域，后续若要子域内直接登录，再单独设计 callback 白名单和 cookie/token 域策略。
- Vercel wildcard domain 需要子域 NS 委派验证成功后才会自动签发证书，Route53 NS delegation 生效可能需要 DNS propagation 时间。