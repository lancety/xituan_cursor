---
name: list-to-detail-smooth-transition
description: >-
  Smooth list→detail native-stack transitions (preview params + press-in
  prefetch + defer below-fold only). Rejects empty-shell-wait-then-fetch.
  Use when Expo/RN merchant (or similar) push detail feels janky, TTI feels
  longer after “shell” fixes, activity/offer/preorder detail navigation,
  onPressIn prefetch, route headerImage/title preview, or transitionEnd deferral.
---

# List → Detail Smooth Transition

**Canonical implementation（merchant Expo）**：活动列表 → `ActivityDetail`

| 文件 | 职责 |
|------|------|
| `xituan_app_merchant/src/utils/activity-detail-prefetch.util.ts` | `prefetch` / `load`（inflight 去重） |
| `xituan_app_merchant/src/utils/navigation-interaction.util.ts` | `runAfterScreenTransition`（`transitionEnd` + timeout） |
| `xituan_app_merchant/src/screens/ActivityDetailScreen.tsx` | 预览上屏 + 详情立刻 load + 订单延后 |
| `MerchantActivityCard` / `PromotionHomeScreen` / `MerchantHomeScreen` | `title`/`headerImage` 入参 + `onPressIn` |

改列表进详情、转场卡顿、或「先空壳再请求」类方案前 **先读本 skill**。

---

## 何时使用

- native-stack 推详情页卡顿 / 白屏跳变 / 输入区或内容「先顶上再下去」
- 用户反馈：过渡平滑了但**看到内容更慢**
- 新做列表→详情（活动、订单、新闻等）要复用同一套手感
- 准备用 `InteractionManager` / `transitionEnd` **整页推迟请求**

---

## 核心思路（一句话）

**列表态即时上屏 + 按下即预取，详情主请求与推页动画并行；只把首屏以下延后。禁止「空壳等动画结束再请求」。**

目标 TTI：`≈ max(动画, 请求)`，而不是 `动画 + 请求`。

---

## 禁止方案

| 方案 | 为何不行 |
|------|----------|
| 轻量空壳 + `transitionEnd` 后再拉**整页**详情 | 平滑但 TTI = 动画 + 网络，用户等更久 |
| 仅靠 `InteractionManager.runAfterInteractions` 等 native-stack 动画 | 经常**不等** slide，不可靠 |
| loading 时用无 `flex:1` 的 spinner **替换**主内容区 | 底栏/composer 被顶到屏幕上方，动画结束再跳布局 |

---

## 设计原子（必须遵守）

### A. 列表预览入路由（第一帧有内容）

| 原子 | 规则 |
|------|------|
| 传参 | 至少 `title` + 头图/`headerImage`（列表已有字段） |
| 详情首帧 | 用 params 画标题/hero；**不要**等 API 才出主视觉 |
| 全页 spinner | 仅当 **无预览且** 尚无 detail：`loading && !detail && !hasPreview` |
| 数据回来 | 可替换为完整 gallery / 文案（允许轻微升级，避免整页闪白） |

### B. 按下预取（网络与手势/动画重叠）

| 原子 | 规则 |
|------|------|
| 时机 | 卡片 `onPressIn`（或等价 press-in）调用 `prefetch` |
| 导航 | `onPress` → `navigate`；**不要**只在详情 `useEffect` 才首次请求 |
| inflight | 同一 `mode:id` 共用一个 Promise；`load` 消费后删除 key；失败删除以便重试 |
| 误触 | 允许少量无效请求；不要为去抖而推迟到动画后 |

### C. 详情加载优先级

| 层级 | 何时请求 |
|------|----------|
| **主详情**（首屏必要） | 进页立刻 `load`（命中 prefetch 则 await 同一 Promise） |
| **首屏以下**（订单列表、重块、次要 Tab） | `navigationInteractionUtil.runAfterScreenTransition` 后再拉 |
| 取消 | focus/unmount 时 `task.cancel()` |

### D. 布局占位（与转场正交但常一起炸）

| 原子 | 规则 |
|------|------|
| 消息/列表区 | 始终占满剩余高度（`flex: 1` / `minHeight: 0`） |
| 底栏 | 钉在 column 底部；loading 用 overlay，**不要**拆掉 flex 子树 |

---

## 实施检查清单

### 路由 / 类型

- [ ] Detail params 含预览字段（如 `title?`、`headerImage?`）
- [ ] 所有入口（首页卡片、营销列表、更多…）都传预览；漏传会退回全页转圈

### 列表卡片

- [ ] `onPressIn` → `prefetch(mode, id)`
- [ ] `onPress` → `navigate` + params（预览字段）
- [ ] 不要在 `onPress` 里再 `setLoading` 挡住列表

### Prefetch util

- [ ] `prefetch` 只点火、不阻塞 UI
- [ ] `load` 复用 inflight；无 inflight 再发请求
- [ ] 失败从 Map 删除

### 详情页

- [ ] 首帧：params 预览 → hero/标题
- [ ] `useFocusEffect` / mount：**立刻** `load` 详情
- [ ] 仅 below-fold 走 `runAfterScreenTransition`
- [ ] 未使用「空壳 + 整页等 transitionEnd」

### 转场 defer util

- [ ] 优先 `navigation.addListener('transitionEnd')`
- [ ] `event.data.closing` 忽略
- [ ] timeout fallback ≈ stack `animationDuration`
- [ ] 提供 `cancel`

---

## 代价（对用户说明时用）

1. **可能白打的请求**：press-in 后滑走仍可能发出  
2. **预览 vs 真数据**：首帧列表态，回来后可能换图/文案  
3. **工程接线**：新入口要接 params + `onPressIn`，否则退回卡顿/转圈  

---

## 常见踩坑

| # | 坑 | 后果 | 对策 |
|---|-----|------|------|
| 1 | 空壳等动画再请求 | 更丝滑但更慢 | 主数据立刻 load + prefetch |
| 2 | 只 defer 整页 | 同 #1 | 只 defer below-fold |
| 3 | 入口漏传 `headerImage` | 仍全屏 spinner | 对齐所有 navigate 点 |
| 4 | 入口漏 `onPressIn` | 动画期间无飞行中请求 | 卡片统一 press-in |
| 5 | `InteractionManager` 当动画完成 | 过早跑重活抢 JS | `transitionEnd` + timeout |
| 6 | loading 替换 flex 列表 | 底栏飞顶 | 保持 flex 区 + overlay |
| 7 | prefetch 不共用 Promise | 双请求竞态闪烁 | inflight Map |

---

## 移植到其它列表→详情

1. 复制「预览 params + prefetch util + load 复用」三件套  
2. 标明 **首屏字段** vs **below-fold**  
3. 用本 checklist 过一遍入口  
4. 不要默认空壳方案；若产品强要求 skeleton，skeleton 只能盖 **未知区**，不能推迟主请求  

---

## 相关

| Skill / 规则 | 关系 |
|--------------|------|
| `ai-coding-principles` | 最小 diff；前端卡顿 ≤2 次代码尝试后再加诊断 log |
| `openim-client-scroll-interaction` | 另一类「进页秒开」：本地缓存预览；本 skill 偏 **无本地缓存时的网络并行** |
| `manual-npm-task-restart` | 改完让开发者 Reload，勿代启 Metro |
