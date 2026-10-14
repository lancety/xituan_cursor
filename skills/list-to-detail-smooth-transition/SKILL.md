---
name: list-to-detail-smooth-transition
description: >-
  Taobao-style navigation data path: first-paint preview/snapshot, press-in
  prefetch, main fetch parallel with push, silent refresh. Does NOT own
  transition-frame render budget. Use for list→detail / list→list TTI,
  onPressIn prefetch, peek/remember hydrate. For slide/jank during animation,
  use page-transition-render-budget.
---

# List → Detail / List 淘宝式切换（数据路径）

**职责边界**：解决「推页时白屏/转圈、主数据等动画结束才请求、二次进入闪骨架」。  
**不负责**：过场动画期间 JS 贴渲染过重导致的卡顿 —— 见 `page-transition-render-budget`。

**Canonical（merchant）**：活动列表 → `ActivityDetail`  
`activity-detail-prefetch.util` + `navigationInteractionUtil` + 预览 params / below-fold defer。

**Canonical（customer 列表→列表）**：Profile → `OrderListScreen`  
`consumer-order-list-prefetch.util`：`peek` / `remember` + 静默刷新。

---

## 核心思路

**列表态/快照即时上屏 + 按下即预取，主请求与推页并行；只把真正 below-fold 延后。禁止「空壳等动画结束再拉主数据」。**

目标 TTI：`≈ max(动画, 请求)`，不是 `动画 + 请求`。

---

## 默认必做（L0）

| # | 做什么 | 典型手段 |
|---|--------|----------|
| A | 首帧有内容 | 详情：路由预览 params；列表：内存 `peek()` hydrate |
| B | 按下预取 | `onPressIn` + inflight 共用 Promise；`delayPressIn={0}` |
| C | 主数据立刻 load | 进页即 `load`（复用 prefetch），**勿**等 `transitionEnd` |
| D | 仅 below-fold defer | `runAfterScreenTransition`（或等价 transitionEnd + timeout） |
| E | 有快照时静默刷新 | 不 `setLoading(true)` 清空已有行 |
| F | loading 不拆 flex 主树 | overlay / ListEmpty / skeleton 盖未知区 |

可选（问一句再做）：冷启动 AsyncStorage 持久化快照；下游详情也套 L0；首屏 N 张 `Image.prefetch`；冷路径短超时 `waitBrief` 后再 navigate。

---

## 禁止

| 方案 | 为何 |
|------|------|
| 空壳 + 动画结束后再拉**整页**主数据 | TTI = 动画 + 网络 |
| 仅靠 `InteractionManager` 当动画完成 | 常不等 native-stack / 路由转场 |
| loading **替换** flex 主内容区 | 底栏/composer 飞顶 |
| 推迟 `onPressIn` 到动画后 | 抵消预取 |

---

## 检查清单

- [ ] 首帧预览或 `peek`；无数据才全屏 spinner
- [ ] 入口均 `onPressIn` → prefetch；`onPress` → navigate
- [ ] 目标页立刻主 `load`；仅 below-fold 走 transitionEnd
- [ ] 有行时静默刷新；prefetch inflight 去重
- [ ] 动画期卡顿另开 `page-transition-render-budget`（埋点 → 改结构/渲染预算 → 清 log）

---

## 相关

| Skill | 关系 |
|-------|------|
| `page-transition-render-budget` | 过场 JS 渲染预算、列表硬切、状态隔离、嵌套简化 |
| `ai-coding-principles` | 最小 diff；前端卡顿 ≤2 次代码尝试后再诊断 log |
| `expo-paginated-list-scroll` | 列表滚动恢复；与进页 TTI 正交 |
