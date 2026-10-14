---
name: page-transition-render-budget
description: >-
  Cross-stack page transition jank: cut first-paint work during route/stack
  animation on RN Expo apps, WeChat miniprogram, CMS/Site/Platform React web.
  Log → analyze render budget (list cap, setState coalesce, state isolation,
  nesting, memo) → fix → remove logs. Builds on list-to-detail-smooth-transition
  data path; use when slide/push animation stutters despite prefetch/peek.
---

# 页面过场动画改进（渲染预算）

跨端：`xituan_app_*`（RN）、`xituan_wechat_app`、`xituan_cms` / `xituan_site` / `xituan_platform`（React）。  
原理相同：**路由/栈过场的若干百毫秒内，JS 线程与布局被首屏提交占满 → 动画掉帧**。

**与淘宝式 skill 的分工**

| Skill | 管什么 |
|-------|--------|
| `list-to-detail-smooth-transition` | 数据路径：peek / press-in prefetch / 主请求并行 / 静默刷新 |
| **本 skill** | 渲染预算：动画期少贴、少 setState、少嵌套、状态隔离、列表硬切 |

淘宝式 **不能单独明显消除** 过场卡顿；有 peek 仍可能因「首帧挂满重列表 + 动画期 setState」掉帧。先保证数据路径，再按本 skill 做渲染侧。

---

## 何时触发

- 用户说：过场卡顿、推页掉帧、slide 不跟手、进页动画抖
- 已做 prefetch/peek 仍卡
- 关键词：过场动画、transition jank、`transitionEnd`、首屏渲染预算、列表硬切

---

## 工作流（必须按序）

### Phase A — 埋点（目标页 + 入口）

在**本次要优化的入口与目标页**加临时诊断 log（统一前缀，便于 Metro / DevTools 过滤）。

建议前缀：`[<ScreenName>:transition]`（例 `[ProductsNav:jank]`）。

最少打点：

| 时机 | 记什么 |
|------|--------|
| 入口 pressIn / navigate | peek 行数、是否 waitBrief |
| 目标页首帧 construct / 首次 render | items 数、loading、关键 chrome 是否展开 |
| 列表 `renderItem` / 行组件 | 累计次数（可对高频 emit 封顶，避免 log 本身抢 JS） |
| 动画期发起的 fetch / setState | categories、bootstrap、enrich、filter expand |
| 转场结束 | `transitionEnd` / 路由 `afterEach` / 小程序 `onReady` 后等价点；summary：各 render 计数、jank 帧 |

RN：可用 rAF 采样（仅记 Δf ≥ ~22ms 的慢帧 + 稀疏 ok 帧）。  
Web：Performance / 长任务或自建 rAF。  
小程序：`setData` 次数与字段体积、首屏节点数。

告知开发者过滤串，请其复现后贴 **session 开始 → transitionEnd summary** 一段。

### Phase B — 对照关注点分析日志

按下方「关注点清单」归因，**只改日志指向的项**（最小 diff）。  
≤2 次无埋点瞎改仍无效时，必须回到 Phase A（与 `ai-coding-principles` 前端诊断一致）。

### Phase C — 开发者确认满意后清 log

删除 util 与全部调用；保留有效结构/预算改动。勿留「以防万一」的 console。

---

## 关注点清单（按收益优先）

### 1. 动画期列表渲染数量（硬切）

| 有效 | 无效 / 慎用 |
|------|-------------|
| **`data` 硬切**前 N 行（N≈首屏，如 10），`transitionEnd` 后再换成全量 | 只调 `initialNumToRender` / `windowSize` —— 仍会继续贴到 16+ |
| 动画期关掉 `onEndReached` / 分页 append | |

小程序：首屏 `wx:for` / 列表只喂前 N 条，再 `setData` 全量。  
Web：虚拟列表首屏 window 收紧，或同上硬切 data。

### 2. 动画期 setState / setData 合并与推迟

- **首帧需要的状态一次到位**（peek hydrate 时 `loading=false` + items 齐），避免 construct 后连环 `setX` → `setY`。
- peek 已与网络结果同 id 时 **skip setItems**（静默确认，不触发列表 reconcile）。
- **非首屏必需**的请求（分类条、次要 Tab、thumb 批量 prefetch、字母表）→ 等转场结束再 fetch / `setData`。
- 主列表数据：淘宝式仍主张与动画并行；若已有 peek 上屏，**解析/map/setState 可延后到 transitionEnd**（网络可提前 inflight）。

### 3. 状态隔离（子更新不打爆父级 / siblings）

- 分类、筛选展开、字母条等 **本地 state 放进 memo 子树**，到达时 **不要** `setState` 在拥有 `FlatList`/`ScrollView` 的父组件上。
- 列表行 `memo`；行内回调 `useCallback` 稳定；避免 renderItem 每帧新闭包导致整表废 memo。
- 小程序：能拆开的 `setData` 路径拆开；避免整页对象每次重设。

### 4. 组件树 / 嵌套（布局计算）

- 过场目标页：**固定顶栏 chrome + 单层列表**，避免 ScrollView 套列表、多层 sticky、过深 View 包裹。
- 行高固定时给 **`getItemLayout` / 等价**，减少 measure。
- 行内：主题 `StyleSheet` 进程级复用；重复 SvgXml / 重图标做 **tint/xml 缓存 + memo**（外观不变，只减 CPU）。

### 5. Lite / 渐进增强（须克制）

| 可用 | 不要默认做 |
|------|------------|
| **首屏以下**或开发者明确接受的骨架区 | 动画期藏掉首屏已可见的图 / 徽章 / 操作，结束后再「补齐」—— 肉眼很明显，体验差（产品列表实践已否决） |
| 永久简化行结构（更少嵌套、更轻控件）且视觉验收通过 | 用明显闪烁换帧率 |

#### 禁止：分帧挂主列表（硬禁）

**禁止**「先只画顶栏 / placeholder，再用双 `requestAnimationFrame` / `listMounted` / 等价手段，等若干帧或等 `transitionEnd` 之后才挂 `FlatList` / 主卡片列表」。

| 为何禁 | 结果 |
|--------|------|
| 主列表是首屏内容，不是次要 chrome | 过场期间空白或 spinner，**动画结束后才突然出卡**——比掉帧更差 |
| 与「硬切 N 行仍首帧上屏」相反 | 硬切是减量；分帧挂载是**零量再补齐** |

允许：`data` 硬切（首帧仍有前 N 卡）；次要条（分类 fetch 等）等 `transitionEnd`。  
不允许：整表挂载推迟到动画后。

### 6. 入口侧（配合淘宝式，本 skill 只点到为止）

- `pressIn` prefetch；冷路径短 `waitBrief`，有 peek 则立刻 navigate。
- 不把「整页主数据」推迟到动画结束（那是淘宝式反模式）。

---

## 实施检查清单

### 开做

- [ ] 已读/对齐 `list-to-detail-smooth-transition` 数据路径（或确认非列表跳转）
- [ ] Phase A 埋点已加；向开发者要 log
- [ ] 用 log 勾选下方改动项，未点名的不大重构

### 常见针对性改动

- [ ] 动画期列表 **data 硬切** N；结束后恢复全量
- [ ] 分类/次要 chrome：子组件隔离 + `transitionEnd` 后再 fetch
- [ ] peek 命中 skipSet；首帧 state 合并
- [ ] 去掉过深嵌套 / 无效 sticky；行 `getItemLayout` + 样式/图标缓存
- [ ] 未采用显眼的首屏 lite 补齐（除非开发者书面接受）
- [ ] **未**分帧推迟挂主 FlatList/卡片（首帧即可有硬切后的列表）

### 收尾

- [ ] 开发者确认手感满意
- [ ] Phase C 删除全部诊断 log / util
- [ ] 未代启长期 Metro/dev server（`manual-npm-task-restart`）

---

## 平台钩子速查

| 栈 | 转场结束信号（示例） | 列表硬切 |
|----|----------------------|----------|
| RN native-stack | `transitionEnd` + timeout（`navigationInteractionUtil`） | `data={animating ? items.slice(0,N) : items}` |
| React Router / 自建转场 | 动画 `onRest` / 固定 duration | 同上或虚拟列表 window |
| 微信小程序 | 路由动画约结束后再 `setData` 扩展列表；或首包只 N 条 | `list.slice(0,N)` 再全量 |

---

## 禁止 / 易错

| 项 | 说明 |
|----|------|
| 只靠淘宝式 prefetch 当「动画就不卡」 | 数据到了仍可能 20 行重卡同时 mount |
| 只调 VirtualizedList 窗口参数 | 必须 **data 硬切** 才稳 |
| **分帧挂主列表**（顶栏先、双 rAF/`listMounted` 后再挂 FlatList） | **硬禁**——见 §5 |
| 动画期父级 setCategories 等 | 易带动 FlatList 父级 re-render |
| 诊断 log 打满每一行 | log 本身制造 jank；封顶或采样 |
| 未确认就清 log | 须开发者满意后再 Phase C |
| 未问就上 FlashList/改导航模型等大方案 | 先穷尽本清单小改 |

---

## Canonical 参考（merchant 产品列表）

实践中验证有效、且无「首屏控件延迟显现」的组合：

1. Home `pressIn` prefetch + 冷 `waitBrief` / 有 peek 立刻 navigate  
2. `peek` hydrate；bootstrap 命中 skipSet；有 peek 时 map/set 可放到 transitionEnd  
3. 分类条独立 memo 子树，`fetchEnabled` 等转场结束  
4. 推页期间 `data` 硬切 `ANIM_FIRST_ROWS`（如 10）  
5. 固定行高 + 主题样式缓存 + 操作图标 SVG tint 缓存  
6. **不做** 首屏图/徽章/操作的 lite 延后显示  

细节以当前 `ProductsListScreen` / `ProductListCard` 为准。

---

## 相关

| Skill / 规则 | 关系 |
|--------------|------|
| `list-to-detail-smooth-transition` | 数据路径 L0；本 skill 接渲染预算 |
| `ai-coding-principles` | 最小 diff；log 辅助诊断 Phase B/C |
| `expo-paginated-list-scroll` | 滚动恢复；与过场硬切正交 |
| `manual-npm-task-restart` | 勿代启 Metro / 各端 dev |
