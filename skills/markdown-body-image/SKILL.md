---
name: markdown-body-image
description: >-
  CMS markdown body images: direct-upload, dw display width, and reader layout
  (container cap, same-line images stay on one row). Use when changing MarkdownEditor
  image insert, body-image pending/commit, or Site, customer app, merchant app, or
  WeChat markdown image display.
---

# 正文 Markdown 图片

权威说明：[devGuide/markdown-body-image.md](../../../xituan_agent/devGuide/markdown-body-image.md)。

## 何时用

- 改 CMS `MarkdownEditor` 插图、裁剪工作室或预览拖拽 `dw`。
- 改 `body_image_pending` / `offer_description_images` / `preorder_description_images`，或新闻正文复用 `news_images`。
- 改 Site、顾客 App、商家 App、微信 wemark 对正文图的展示。

## 当前契约

1. 只有传入 `imageUpload` 的 CMS 编辑器才上传。语法是 `![alt](path "dw=480")`。
2. 直传 `waitNormalize: false`、`retainPending: true`。编辑器看大图；阅读端按 `dw` 选 `_w*`，失败回原图。
3. 阅读端：宽和高都不超过容器 100%。同一行、中间没有换行的图按各自 `dw` 从左贴齐；超出容器才一起缩小，不拆行。
4. 商家 App 图片按钮仍插入 URL 占位。微信商户团购 / 预定描述仍是 textarea。不要在这两处接上传，除非在做 [移动端插图任务](../../../xituan_agent/devGuide/planned-work/entries/2026-11-mobile-markdown-body-image-insert.md)。
5. 最小 / 最大宽度和图组设置还在 planned plan，不要提前写成 `min` / `max` / `align`。

## 调用方

| 页面 | bind |
|------|------|
| CMS 团购描述 | `OFFER_DESCRIPTION_IMAGES`，无 id 时 `BODY_IMAGE_PENDING` |
| CMS 预定描述 | `PREORDER_DESCRIPTION_IMAGES`，无 id 时 `BODY_IMAGE_PENDING` |
| CMS 新闻正文 | 有 id 时 `NEWS_IMAGES` 且 `retainPending: true`；无 id 时 `BODY_IMAGE_PENDING` |
