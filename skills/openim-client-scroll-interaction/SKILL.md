---
name: openim-client-scroll-interaction
description: >-
  OpenIM chat room inverted scroll (rotateX), enter/leave position restore,
  history pagination guards, cache snapshot design, and merchant-staff chat
  entry (merchantId in navigate URL). Use when changing WeChat packageIm chat
  room, merchant panel chat navigation, OpenIM client scroll/history UX, or
  porting the same patterns to CMS, Site, or Platform chat UIs.
---

# OpenIM Client Scroll Interaction

**Canonical doc（中文全文）**：`xituan_agent/devGuide/openim/chat-room-inverted-scroll-design.md`

改动 OpenIM 客户端交互前 **先读 devGuide**。本 Skill 只保留设计原子、标准与踩坑 checklist。

---

## 何时使用

- 修改 `packageIm/pages/chat/room`、倒置滚动、进房/离房位置
- 新增或修改商户侧（我的网店 / 活动订单）跳转 OpenIM 聊天室
- 改 `openim-history-loader`、`chat-room-inverted-scroll.util`
- 改 `xituan_codebase` 中 `openim-chat-scroll-intent`、`openim-history-sync`、消息缓存 meta
- 在 CMS / Site / Platform 做聊天室或类 IM 消息列表

---

## 核心思路（一句话）

**用 rotateX 倒置让视觉底部 = scrollTop≈0，从而贴底零滚动；位置恢复用单次 scroll-top + 缓存三元组；加载更早走 scrolltolower 并在程序化滚动期间加 guard。**

---

## 设计原子（必须遵守）

### A. 布局与坐标

| 原子 | 规则 |
|------|------|
| 倒置 | `scroll-view` `rotateX(180deg)` + 每行 `rotateX(180deg)` |
| 视觉底部 | `scrollTop ≤ 80px` → `stickToBottom = true` |
| 视觉顶部 | `scrollTop ≥ maxTop - threshold` → 可加载更早 |
| 禁用 | 恢复位置用 `scroll-into-view`；用 `scrolltoupper` 加载更早 |

### B. 数据序

| 存储 | 顺序 |
|------|------|
| `messages` 内存 | seq **升序** |
| `timelineEntries` 渲染 | **reverse** 后展示 |

### C. 离房快照（cache meta）

| 字段 | 写入规则 |
|------|----------|
| `stickToBottom` | 由 `isAtBottom(scrollTop)` |
| `lastScrollTop` | 贴底写 `0`；否则当前 `scrollTop` |
| `lastViewSeq` | viewport **屏幕最上方**可见行 seq（`pickViewportAnchorFromMessageRows`） |

### D. 进房意图（二选一）

| kind | 条件 |
|------|------|
| `bottom` | 空列表；或 `lastViewSeq >= msgMax` 且非强制离底 |
| `anchor` | `lastViewSeq` 命中消息；或 **未读** `unreadAnchorSeq` |

### E. 历史与分页

| API | 用途 |
|-----|------|
| 默认 `limit` | 最新一页 |
| `beforeSeq` | 用户上拉更早 |
| `afterSeq` | 本地 `cachedMaxSeq` 之后增量（可选，老端不传） |

### F. Guard（程序化滚动期间）

以下任一为 true → **禁止** `loadOlderHistory` / `onScrollToOlder`：

- `initialHistoryLoading`
- `enterScrollRestoreInFlight`
- `programmaticScrollInFlight`

### G. 未读 vs 滚动恢复（分路）

| 场景 | 允许 `loadOlderForAnchor` |
|------|---------------------------|
| `hasUnreadAnchor && anchorSeq` 且消息不在窗口 | ✅ |
| 仅恢复 `lastViewSeq` / `lastScrollTop` | ❌ |

### H. 商户侧进房路由（MERCHANT_STAFF）

| 原子 | 规则 |
|------|------|
| 必带 query | `conversationId` + `title` + `channel=merchant_staff` + **`merchantId`** |
| merchantId 来源 | `getCachedMerchantId()` → `merchantPanelAccessUtil.readCache()?.merchantId` |
| 统一入口 | `merchantPanelStaffChatNavWechatUtil.buildStaffChatRoomUrl` — **禁止**手写 room URL |
| 原因 | `tryApplyInstantCachePreview` 在 `bootstrapRoom` 开头需 `buildMerchantStaffKey(mid, userId)`；缺 mid 则 instant cache 永不命中 → 每次全屏「正在同步会话...」 |

**devGuide 全文：** `chat-room-inverted-scroll-design.md` §5.4

---

## 实现检查清单

### 布局 / WXML

- [ ] `message-area-inverted` + `message-row-inverted` 成对存在
- [ ] `bindscrolltolower`（非 upper）+ `lower-threshold`
- [ ] `scroll-top="{{enterAnchorScrollTop}}"` 单次恢复
- [ ] 恢复前 `messageAreaScrollReady: false` + `opacity: 0`（可选但推荐）

### room.ts 生命周期

- [ ] `onHide` → probe viewport → `persistChatScrollSnapshot`
- [ ] `tryApplyInstantCachePreview` 有 `lastScrollTop` 时同批写 `enterAnchorScrollTop`
- [ ] 历史完成后 `applyEnterAnchorAfterLayout` → verify
- [ ] **不**将 `enterAnchorScrollTop` 恢复后清为 `''`

### 滚动事件

- [ ] `onMessageScroll` 更新 `lastMessageScrollTop` + `stickToBottom`
- [ ] `isNearVisualTop` 兜底触发 load older
- [ ] `scheduleLoadOlderOnReachVisualTop` debounce

### 历史 bootstrap

- [ ] `loadEnterRoom` + 可选 `afterSeq` 增量
- [ ] instant cache 时 `canIncrementalSyncFromCache` / `syncUiMessagesIncremental`
- [ ] `loadOlderForAnchor` 仅在 `hasUnreadAnchor`

### 商户侧进房导航

- [ ] MERCHANT_STAFF 跳转使用 `merchantPanelStaffChatNavWechatUtil.buildStaffChatRoomUrl`
- [ ] URL 含 `merchantId`（非等 `getConversation` 后再补）
- [ ] 新 call site 不复制 URL 字符串

### 共享层（codebase）

- [ ] meta 含 `stickToBottom` / `lastScrollTop` / `lastViewSeq`
- [ ] `openimChatScrollIntentUtil.resolveScrollIntent` 未读与 lastView 优先级正确
- [ ] 改契约时按 `xituan-codebase-change-scope` 评估消费者

### 跨端（CMS / Site / Platform）

- [ ] 复用「双序模型 + 快照三元组 + 意图二分 + guard」
- [ ] 端上 adapter 实现缓存读写，逻辑放 codebase
- [ ] 明确该平台 scroll 容器的「底部/顶部」与 max scroll 语义

---

## 常见踩坑

| # | 坑 | 后果 | 对策 |
|---|-----|------|------|
| 1 | 倒置后用 `scrolltoupper` 加载更早 | 滚顶无请求；恢复时误拉全历史 | 用 `scrolltolower` + guard |
| 2 | 恢复大 scrollTop 无 guard | 连锁 `beforeSeq` API | 三 flag 拦截 `loadOlder` |
| 3 | probe 取 DOM 第一条为 anchor | `lastViewSeq` 恒为最新 | 按 `top` 取 viewport 最上行 |
| 4 | `scroll-into-view` 恢复 | 滚到 0 或无效 | 仅 `scroll-top` + verify |
| 5 | 未读 anchor 与 scroll 恢复混用 | 进房拉全量历史 | `loadOlderForAnchor` 仅 unread |
| 6 | 首帧渲染后异步 scroll | 闪到底部 | instant cache 同批 scroll-top + opacity |
| 7 | `enterAnchorScrollTop` 置空释放 | 回到视觉底部 | 保持数值，勿设 `''` |
| 8 | 整表 replace 消息 | 列表跳动 | incremental sync from cache |
| 9 | 后端先上、新 `afterSeq` | 老小程序不传 → 无影响 | 可选参数，向后兼容 |
| 10 | MERCHANT_STAFF 跳转缺 `merchantId` | 每次进房全屏 loading；有缓存也不秒开 | 用 `merchant-panel-staff-chat-nav.wechat.util` 带 merchantId |

---

## 设计标准

1. **单一滚动真相**：rotateX 下以 `scrollTop` 为准；`stickToBottom` 由阈值派生，不单独猜测。
2. **一次恢复**：进房 anchor 只走一条链（cached scrollTop → 几何 fallback），避免多路径竞态。
3. **快照可验证**：`verifyRotateXScrollTop` 失败才重试，成功即 `completeEnterScrollRestore`。
4. **日志可 grep**：`[ChatRoom:scroll-restore]` + phase 名；新逻辑延续此前缀。
5. **共享逻辑上浮**：意图、历史同步、merge、cache meta 在 `xituan_codebase`；端上只保留平台 scroll util。
6. **API 向后兼容**：`beforeSeq` / 默认分页保持；`afterSeq` 仅新端使用。
7. **不重建补偿栈**：禁止恢复 `chat-room-scroll-position` 类多层 scrollTop 补丁。

---

## 源码入口

| 文件 | 职责 |
|------|------|
| `packageIm/pages/chat/room/room.ts` | 页面主逻辑 |
| `packageIm/utils/chat-room-inverted-scroll.util.ts` | 倒置坐标 util |
| `packageIm/utils/openim-history-loader.wechat.util.ts` | 微信缓存/历史 |
| `xituan_codebase/utils/openim-chat-scroll-intent.util.ts` | bottom/anchor 意图 |
| `xituan_codebase/utils/openim-history-sync.util.ts` | enter/older/afterSeq |
| `packageMerchant/utils/merchant-panel-staff-chat-nav.wechat.util.ts` | 商户侧 room URL + merchantId |
| `packageIm/lib/openim-identity-session.wechat.util.ts` | channel → identity key（MERCHANT_STAFF 需 merchantId） |

---

## 相关 Skill

| Skill | 关系 |
|-------|------|
| `business-error-line-errors` | 发消息/校验错误展示（非滚动） |
| `xituan-codebase-change-scope` | 改共享类型/util 时的消费者范围 |
| `post-deploy-ledger` | 若 API/缓存格式破坏已发布端，需分阶段 |
