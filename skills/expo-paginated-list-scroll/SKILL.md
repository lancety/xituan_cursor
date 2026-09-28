---
name: expo-paginated-list-scroll
description: >-
  Expo/RN merchant & customer paginated lists: FlatList virtualization,
  onEndReached infinite load, preserve scroll when returning from detail,
  forbid focus replace-reload. Use when building or fixing list→detail
  scroll jump, long product/order/partner/invoice lists, ScrollView+map
  lists, load-more buttons, or useFocusEffect page-1 replace on blur return.
---

# Expo Paginated List + Scroll Restore

**Applies to:** `xituan_app_merchant`, `xituan_app_customer` (all feature lists that push a detail/editor and come back).

**Canonical helpers（两 App 同名同契约）**

| 文件 | 职责 |
|------|------|
| `src/utils/expo-paginated-list.util.ts` | FlatList 性能默认 props、`onEndReachedThreshold`、offset store |
| `src/utils/use-expo-list-scroll-restore.util.ts` | `listKey` 维度保存/恢复 `contentOffset.y` |

改列表、分页、「从详情返回跳顶」、或把 `ScrollView`+`map` 当长列表前 **先读本 skill**。

Related（不替代本 skill）：列表→详情转场手感见 `list-to-detail-smooth-transition`；IM 倒置列表见 `openim-client-scroll-interaction`。

---

## 何时使用

- 新建或改造：产品 / 订单 / 合作方 / 发货单·结算单 / 活动订单 等可分页列表
- 用户反馈：进详情再返回 **滚动位置丢失**、或长列表卡顿
- 仍用 `ScrollView` + `.map` 渲染行，或底部要点「更多」才加载
- `useFocusEffect` 里每次 focus 都 `load(page: 1, replace)`

---

## 核心思路（一句话）

**FlatList 虚拟化 + `onEndReached` 触底加载；native-stack 保活下列表状态不因返回而 replace；再用 `listKey` offset 兜底。禁止用 ScrollView 堆全量行，禁止返回时整表重拉导致跳顶。**

业界默认（RN）：`FlatList`（本仓不强制 FlashList，未引入则勿新加依赖）；触底分页；返回详情依赖 **不卸载 + 不破坏数据**，offset store 防 remount。

---

## 必须遵守

### A. 列表容器

| 规则 | 说明 |
|------|------|
| 用 `FlatList` | 纵向可分页 / 可能很长的业务列表 |
| 禁止 | `ScrollView` + `items.map` 渲染业务行（短静态区块除外） |
| 性能 props | 展开 `expoPaginatedListUtil.listPerfProps` |
| 触底 | `onEndReached` + `expoPaginatedListUtil.endReachedThreshold`；底部仅 `ActivityIndicator`，**不要**可点「更多」 |

### B. 从详情返回 → 回到点击时的位置

| 规则 | 说明 |
|------|------|
| Stack | 保持 native-stack；列表屏 `freezeOnBlur: true`（已有则勿关） |
| **禁止** | focus 回来就 `load({ page: 1 })` / `replace` 整表（这是跳顶主因） |
| 允许首次加载 | mount 或 **第一次** focus 拉第 1 页；筛选项变化、下拉刷新再 replace |
| Offset 兜底 | `useExpoListScrollRestore(listKey)`：blur 写入 store，focus `scrollToOffset` |
| `listKey` | 稳定且区分实例，如 `partner-documents:invoice:all`、`products-list`、`orders:${merchantId}` |
| 筛选项变更 | `expoListScrollOffsetStore.clear(listKey)` 后再从第 1 页加载 |

### C. FlatList 不要被条件卸载

| 错误 | 正确 |
|------|------|
| `loading && empty ? <Spinner/> : <FlatList/>` 且返回时仍 replace 数据 | 有数据时始终挂载同一 `FlatList`；空态用 `ListEmptyComponent`；首次无数据可 spinner |

### D. 例外

| 场景 | 处理 |
|------|------|
| OpenIM 会话/消息 | 跟 `openim-client-scroll-interaction`，不套本 skill 的 offset 约定 |
| 首页「最近 N 条」预览 | 可短列表；点「更多」进的全量页必须遵守本 skill |
| 横向 chip / 轮播 | 可用横向 `ScrollView` / `FlatList`，不强制本 skill 的 y-offset |

---

## 实现清单（新列表或改造）

1. `FlatList` + `keyExtractor` + `renderItem` + `onEndReached`（guard：`loading` / `loadingMore` / `!hasMore`）
2. `...expoPaginatedListUtil.listPerfProps` + `onEndReachedThreshold={expoPaginatedListUtil.endReachedThreshold}`
3. `const scroll = useExpoListScrollRestore('your-stable-key')` → `ref={scroll.listRef}` `onScroll={scroll.onScroll}` `scrollEventThrottle={scroll.scrollEventThrottle}`
4. 数据加载：首次 / 筛选 / pull；**不要**在每次 `useFocusEffect` 里 replace
5. 编辑保存后若必须刷新：局部 patch 行，或显式 `clear(listKey)` + reload（不要静默每次返回都 reload）

---

## 禁止方案

| 方案 | 为何不行 |
|------|----------|
| `ScrollView` + 全量 `map` | 长列表内存与 JS 线程差 |
| 底部按钮「加载更多」作唯一分页入口 | 体验差；应触底自动加载 |
| 每次从详情返回 `load(1, replace)` | 跳顶 + 闪烁 |
| 无 `listKey` 的全局单一 offset | 多列表互相污染 |
| 为「返回保位置」改用 `detachInactiveScreens: false` 全家关掉虚拟化 | 错方向；先修 focus reload |

---

## Authority

全仓清单：`xituan_agent/devGuide/agent-global-dev-rules-checklist.md` §9 领域专项。
