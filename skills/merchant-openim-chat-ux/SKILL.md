---
name: merchant-openim-chat-ux
description: >-
  Merchant Expo OpenIM chat: cross-tab ChatRoom navigation (external vs Messages
  stack), local message cache (engine + RN AsyncStorage adapter), and chat room
  layout so composer stays bottom-docked on first paint. Use when changing
  merchant Messages tab, Home/Promotion open-client-chat, ExternalChatRoute,
  merchantChatNavUtil, ChatRoomScreen, openim-history-loader.merchant,
  message cache storage, or composer/input stuck at top / chat pollutes Messages.
---

# Merchant OpenIM Chat UX

**Scope**：`xituan_app_merchant` Expo 消息 / 跨 Tab 进房 / 本地消息缓存 / 聊天室布局。  
微信 invert 滚动与 scroll 快照见 **`openim-client-scroll-interaction`**（本 skill 不重复 rotateX）。

改动下列能力前 **先读本 skill**：

| 能力 | Canonical 入口 |
|------|----------------|
| 跨 Tab 导航 | `merchant-chat-nav.util.ts`、`MessagesStackRoutes.tsx`、`MerchantMainTabs.tsx` |
| 进房历史 + 缓存 | `openim-history-loader.merchant.util.ts`、`ChatRoomScreen.tsx` |
| RN 存储适配 | `openim-message-cache-storage.rn.util.ts` |
| 共享引擎/淘汰 | `xituan_codebase` → `openim-message-cache-engine` / `openim-message-cache-policy` / `openim-history-sync` |
| 会话 stash | `conversation-nav.util.ts`（仅内存摘要，**不是**消息缓存） |

---

## 一、跨 Tab 聊天导航

### 核心规则（产品）

1. **消息 Tab 自有栈**：`ConversationList` ↔ `ChatRoom`（`MessagesChatRoute`）。切到其它主 Tab 再回消息 → **仍停在当前客户聊天**（保留栈）。
2. **其它入口独立房**：首页头像、订单「聊天」等在 **当前栈** 推 `ChatRoom`（`ExternalChatRoute`），**禁止** `navigate(Messages, { screen: 'ChatRoom' })` 污染消息栈。
2b. **进房隐藏主 Tabs**：Home / Messages / Promotion 栈顶为 `ChatRoom` 时 `tabBarStyle: { display: 'none' }`（`merchantChatNavUtil.isChatRoomFocused`）。
3. **外部房仍打开时点「消息」** → 强制进 **会话列表**（不是外部那个房）。
4. **从消息列表点进任意客户后**，再回到之前有外部房的主 Tab → **pop 外部 `ChatRoom`**，回到进房前一页。
5. 外部房打开时切到营销/更多等（非消息）→ **外部房保持**；仅「去消息 Tab」例外（见 3）。

### 路由注册

| 栈 | 组件 | 说明 |
|----|------|------|
| Messages | `MessagesChatRoute` | 不消费 dismiss flag |
| Home / Promotion | `ExternalChatRoute` | focus 时 `consumeDismissExternalChat` → `goBack` |
| 类型 | `tHomeStackParamList` / `tPromotionStackParamList` 含 `ChatRoom` | 与 Messages 同参：`conversationId` + `title?` |

### `merchantChatNavUtil` 原子

| API | 何时调用 |
|-----|----------|
| `markMessagesListChatOpened()` | 消息列表 `onOpenChat` |
| `clearDismissExternalChat()` | 首页/订单等 **打开外部房之前** |
| `consumeDismissExternalChat()` | `ExternalChatRoute` `useFocusEffect`；true → `goBack` 一次 |
| `hasExternalChatOpen(tabState)` | Messages `tabPress`：其它 Tab 顶路由是 `ChatRoom` |

### Messages `tabPress`

```
若当前已在 Messages → 不拦截（保持列表或房）
若 hasExternalChatOpen → preventDefault + navigate(Messages, { screen: ConversationList })
否则 → 默认，恢复 Messages 栈原样（可回到消息内聊天）
```

### 禁止

| 禁止 | 原因 |
|------|------|
| 首页 `navigate(MESSAGES, { screen: ChatRoom })` | 锁死消息 Tab；返回像回首页 |
| Messages `tabPress` **总是** reset 到列表 | 打断「回消息继续聊」 |
| 外部房与消息房共用同一栈历史 | 互相污染 |

### 新入口接线

1. `conversationNavUtil.stash(conversation)`
2. `merchantChatNavUtil.clearDismissExternalChat()`
3. **当前栈** `navigation.navigate('ChatRoom', { conversationId, title })`
4. 若新主栈也要外部房：注册 `ExternalChatRoute`，勿复用污染 Messages 的写法

---

## 二、本地消息缓存

### 目标

进房 **先画缓存**，再 `afterSeq` / 最新页增量同步；收发写入缓存。体量用 **共享淘汰策略**（条数 + 活跃度），**不用按天 TTL**。

### 政策常量（codebase，勿在端上另造）

| 常量 | 值 | 含义 |
|------|-----|------|
| 历史页 `PAGE_SIZE` | **50** | 进房默认最新一页 |
| `OPENIM_MESSAGE_CACHE_MIN_PER_RECENT_CONVERSATION` | **30** | 近 50 会话至少保留 |
| `OPENIM_MESSAGE_CACHE_RECENT_CONVERSATION_COUNT` | **50** | 受保护会话数 |
| `OPENIM_MESSAGE_CACHE_GLOBAL_BASE_LIMIT` | **5000** | 全局基数 |
| `OPENIM_MESSAGE_CACHE_PER_CONVERSATION_BUFFER` | **30** | 总上限 ≈ `5000 + 30×会话数` |

淘汰：先删不活跃整会话，再删更老消息；近 50 会话保底 30 条。

### RN 适配要点

- 引擎 adapter **同步** `readJson` / `writeJson` / `removeKey`
- RN：`openimMessageCacheStorageRnUtil` = **内存 Map** + `hydrateScope` 读 AsyncStorage + 异步 flush
- 进房前：`ensureReady(scopeKey)` → hydrate → 再 `readCachedUiMessages` / `loadEnterRoom`

### scopeKey（merchant）

稳定用登录身份，**不要**用会晚到的 openimUserId 导致缓存分裂：

`merchant:{merchantId}:staff:{userId}`

### `ChatRoomScreen` 进房顺序

1. `ensureReady(scopeKey)`
2. `readCachedUiMessages` → 有则立刻 `setBubbles` 且 `setLoading(false)`
3. `loadEnterRoom`（有缓存时用假 `maxSeq` 走 `afterSeq` 增量；见 loader）
4. 收发：`appendUiMessages` / `persistBubbles`

### API

- `fetchMessages(id, limit, beforeSeq?, afterSeq?)` — 增量必须支持 `afterSeq`
- 复用 `openimHistorySyncUtil.loadEnterRoom`，勿平行实现第二套 merge

### 禁止

| 禁止 | 原因 |
|------|------|
| 只每次 API、无 instant cache | 进房必转圈 |
| 端上自创按天清库忽略 policy | 与微信/共享层不一致 |
| 未 hydrate 就 sync 读 adapter | 空缓存误判 |
| 把 `conversationNavUtil` 当消息存储 | 只有会话摘要 |

---

## 三、输入栏 / 消息区布局（首屏贴底）

### 根因（已修过的坑）

`loading` 时用无 `flex:1` 的 `ActivityIndicator` **替换**整个消息区 → column 里 composer 贴在 header 下 → **输入框在屏幕上方**；列表挂上后再跳底。

### 必须布局

```
View (flex:1)
  header (shrink)
  View.messageArea (flex:1, minHeight:0)
    FlatList (flex:1, inverted, …)
    overlay spinner（仅无缓存冷启动）
  error?
  View (paddingBottom: keyboardLift)
    ChatComposer
```

| 原子 | 规则 |
|------|------|
| 消息区 | 始终挂载；`flex:1` + `minHeight:0` |
| loading | **overlay**，不拆掉 FlatList |
| composer | 兄弟节点钉在底部，不放进非 flex 占位树 |
| 键盘 | 现有 `useKeyboardBottomInset` / composerLift；面板打开时可 lift=0 |

### inverted 列表

- 数据：内存升序 bubbles → `invertedData = reverse` + `FlatList inverted`
- 空列表 + flexGrow 仍占满中间 → composer 仍在底

---

## 实施检查清单

### 导航

- [ ] 外部入口不写 `MESSAGES + ChatRoom`
- [ ] Home/Promotion 注册 `ExternalChatRoute`
- [ ] 列表开房 `markMessagesListChatOpened`
- [ ] 外部开房前 `clearDismissExternalChat`
- [ ] Messages `tabPress` 仅在 `hasExternalChatOpen` 时去列表

### 缓存

- [ ] hydrate → instant paint → loadEnterRoom
- [ ] scopeKey 稳定（merchant+staff）
- [ ] afterSeq 接线；收发写缓存
- [ ] 淘汰走 codebase policy

### 布局

- [ ] messageArea `flex:1`；无「spinner 替换列表」
- [ ] 冷启动无缓存才全屏 overlay；有缓存不挡 composer

---

## 常见踩坑

| # | 坑 | 后果 | 对策 |
|---|-----|------|------|
| 1 | 跨 Tab 推进 Messages 房 | 消息 Tab 锁客户；返回乱 | 外部栈 `ExternalChatRoute` |
| 2 | tabPress 总 reset 列表 | 回消息不能续聊 | 仅 external 打开时去列表 |
| 3 | 忘记 mark/clear/consume | 外部房不 pop 或误 pop | 按原子表接线 |
| 4 | loading 替换 FlatList | 输入框在顶部 | flex 区 + overlay |
| 5 | 引擎当 async storage 用 | 读空/竞态 | 内存 sync + hydrate |
| 6 | scopeKey 用晚到的 openim id | 缓存 miss / 分裂 | merchant+staff 身份键 |

---

## 相关 Skill

| Skill | 关系 |
|-------|------|
| `openim-client-scroll-interaction` | 微信 invert 滚动、scroll 快照三元组、商户员工进房 URL |
| `list-to-detail-smooth-transition` | 列表→详情预览+预取（非 IM） |
| `manual-npm-task-restart` | 改完让开发者 Reload，勿代启 Metro |
