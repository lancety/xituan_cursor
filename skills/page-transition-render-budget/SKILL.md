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

## 通用有效方案（跨功能页可借鉴、默认无实质副作用）

以下从活动详情等页沉淀，**适用于列表页 / 详情页 / 混合页**，作为默认工具箱。  
「无副作用」指：不藏首屏已可见内容、不事后突然补控件、不把主列表推迟到动画后；仅减量/推迟非必需工作。

| 方案 | 作用 | 适用面 | 注意（仍算可接受） |
|------|------|--------|-------------------|
| **`runAfterScreenTransition`（或栈等价）解锁 below-fold** | 统一「动画结束」闸门 | 全栈 RN；Web/小程序用等价信号 | timeout fallback；callback 的 source 参数可选 |
| **列表 data 硬切 + 结束后全量（可再 +1 帧 soft reveal）** | 过场少挂行/卡 | 任何首屏列表/订单/商品行 | 结束后短暂仍是前 N 行再扩到全量 |
| **昂贵子树 `memo` 隔离** | 兄弟区 setState 不拖整树 | Hero vs 列表、筛选 vs 正文 | props 引用要稳，否则 memo 失效 |
| **昂贵纯展示叶 `memo`（Markdown/富文本/图廊等）** | 父级 chrome setState 少重 parse/layout | 任意重叶子 | 同上 |
| **指纹 / 同内容 skip `setState`** | 过场少一次无意义 reconcile | 任意详情 hydrate / 静默确认 | 指纹字段须覆盖真实变更 |
| **推迟 measure / onLayout 驱动的 setState** | 过场少一次布局回写 | 任意「量高再判定」UI | 动画期用启发式即可；结束后再精确量 |
| **下方 chrome 占位（peek/preview 已上、detail 未到）** | 过滤条/Tab 不顶到标题下再跳 | list→detail、预览标题已有 | 占位高度接近终态；勿用空盒冒充正文内容 |
| **溢出区用 `maxHeight` 而非固定 `height`** | 短内容不撑出大空白 | 任意收起/封顶区域 | 封顶比例按页设计，非本 skill 定数 |

**不要**当作全站默认、仅在「重 Markdown / 大表格描述」等场景再考虑（有内容截断语义）：

| 特化方案 | 为何不通用 |
|----------|------------|
| 收起态按行截断 MD、展开才全文 | 改变可见文案与表格完整性；非列表页无此结构 |
| 「展开」条与 lite 正文同步 | 依赖截断收起模型；通用原则见下节「避免事后弹出」 |
| 过场挂 lite → 结束后换全文（同树二次 parse） | 已否决：双重 layout 更卡 |
| 空盒 / 源文本冒充 MD、固定 `%` height 撑满 | 已否决：观感或空白差 |

**通用 UX 推论（从特化里抽出、无领域绑定）：**

- 若某控件与首屏正文应一起被看到，**与正文同帧挂**，不要等 `transitionEnd` 再弹出（避免「动画结束突然多一块」）。
- 减负靠 **减节点 / 推迟非首屏工作**，不靠 **藏已可见内容再补齐**。

---

## 关注点清单（按收益优先）

### 1. 动画期列表渲染数量（硬切）

| 有效 | 无效 / 慎用 |
|------|-------------|
| **`data` 硬切**前 N 行（N≈首屏，如 5–10），`transitionEnd` 后再换成全量 | 只调 `initialNumToRender` / `windowSize` —— 仍会继续贴到 16+ |
| 结束后可 **再等 1 帧**（`ordersFullReveal` / 等价）再挂全量，避免与 unlock 同帧抢布局 | |
| 动画期关掉 `onEndReached` / 分页 append | |

小程序：首屏 `wx:for` / 列表只喂前 N 条，再 `setData` 全量。  
Web：虚拟列表首屏 window 收紧，或同上硬切 data。

### 2. 动画期 setState / setData 合并与推迟

- **首帧需要的状态一次到位**（peek hydrate 时 `loading=false` + items 齐），避免 construct 后连环 `setX` → `setY`。
- peek / 详情已与网络结果同 id（或 **内容指纹** 相同）时 **skip setItems / setDetail**（静默确认，不触发 reconcile）。
- **非首屏必需**的请求（分类条、次要 Tab、thumb 批量 prefetch、字母表、订单全量）→ 等转场结束再 fetch / `setData`。
- 主列表数据：淘宝式仍主张与动画并行；若已有 peek 上屏，**解析/map/setState 可延后到 transitionEnd**（网络可提前 inflight）。
- **measure / onLayout → setState**（量高判定展开、吸顶阈值等）：动画期可跳过或只用启发式，**unlock 后再精确 measure**。

### 3. 状态隔离（子更新不打爆父级 / siblings）

- 分类、筛选展开、字母条、订单区等 **本地 state 放进 memo 子树**，到达时 **不要** `setState` 在拥有 `FlatList`/`ScrollView` / 重 Hero 的父组件上。
- 列表行 `memo`；行内回调 `useCallback` 稳定；避免 renderItem 每帧新闭包导致整表废 memo。
- 重叶子（Markdown、大图廊、复杂表）单独 `memo`，父级仅 chrome setState 时跳过重 parse。
- 小程序：能拆开的 `setData` 路径拆开；避免整页对象每次重设。

### 4. 组件树 / 嵌套（布局计算）

- 过场目标页：**固定顶栏 chrome + 单层列表**，避免 ScrollView 套列表、多层 sticky、过深 View 包裹。
- 行高固定时给 **`getItemLayout` / 等价**，减少 measure。
- 行内：主题 `StyleSheet` 进程级复用；重复 SvgXml / 重图标做 **tint/xml 缓存 + memo**（外观不变，只减 CPU）。
- 需要高度封顶的区域：优先 **`maxHeight` + overflow hidden**，避免短内容被固定 `height` 撑出空白。

### 5. Lite / 渐进增强（须克制）

| 可用 | 不要默认做 |
|------|------------|
| **首屏以下**或开发者明确接受的骨架区；**peek 下预留下方 chrome 槽位**（高度接近终态） | 动画期藏掉首屏已可见的图 / 徽章 / 操作 / 展开条，结束后再「补齐」—— 肉眼很明显（已否决） |
| 永久简化行结构（更少嵌套、更轻控件）且视觉验收通过 | 用明显闪烁换帧率 |
| 空盒 / 纯源文本冒充富文本正文「先顶住布局」 | 同上；与「同帧可见」冲突 |

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

### 7. 附录：重 Markdown / 可折叠长文（非默认）

仅当目标页确有 **GFM 表 / 长 MD 收起** 且日志证明 Yoga 在 layout 屏外节点时：

- 收起态 **按行截断** 再挂 `MarkdownBody`（接受残表）；**展开才挂全文**（不要在 `transitionEnd` 自动换全文 → 二次 parse）。
- 展开条与截断正文 **同帧**；`transitionEnd` 只开 measure。
- 仍遵守上文通用表；空盒、源文本、固定 `%` height、过场 lite→结束后全文 **禁止**。

---

## 实施检查清单

### 开做

- [ ] 已读/对齐 `list-to-detail-smooth-transition` 数据路径（或确认非列表跳转）
- [ ] Phase A 埋点已加；向开发者要 log
- [ ] 用 log 勾选下方改动项，未点名的不大重构

### 常见针对性改动（通用优先）

- [ ] 动画期列表 **data 硬切** N；结束后（可 +1 帧）恢复全量
- [ ] `runAfterScreenTransition`（或等价）作 below-fold / 次要 fetch 闸门
- [ ] 分类/订单/Hero 等：**memo 子树隔离** + 重叶子 memo
- [ ] peek / 详情 **指纹 skipSet**；首帧 state 合并
- [ ] 推迟动画期 **measure→setState**；下方 chrome **占位**防跳动
- [ ] 封顶区域优先 **maxHeight**（非固定 height）
- [ ] 去掉过深嵌套 / 无效 sticky；行 `getItemLayout` + 样式/图标缓存
- [ ] 未采用显眼的首屏 lite 补齐 / 事后弹出控件（除非开发者书面接受）
- [ ] **未**分帧推迟挂主 FlatList/卡片（首帧即可有硬切后的列表）
- [ ] （可选）重 MD 收起：按行截断 + 展开全文；见 §7

### 收尾

- [ ] 开发者确认手感满意
- [ ] Phase C 删除全部诊断 log / util
- [ ] 未代启长期 Metro/dev server（`manual-npm-task-restart`）

---

## 平台钩子速查

| 栈 | 转场结束信号（示例） | 列表硬切 |
|----|----------------------|----------|
| RN native-stack | `transitionEnd` + timeout（`navigationInteractionUtil.runAfterScreenTransition`） | `data={animating ? items.slice(0,N) : items}` |
| React Router / 自建转场 | 动画 `onRest` / 固定 duration | 同上或虚拟列表 window |
| 微信小程序 | 路由动画约结束后再 `setData` 扩展列表；或首包只 N 条 | `list.slice(0,N)` 再全量 |

---

## 禁止 / 易错

| 项 | 说明 |
|----|------|
| 只靠淘宝式 prefetch 当「动画就不卡」 | 数据到了仍可能 20 行重卡同时 mount |
| 只调 VirtualizedList 窗口参数 | 必须 **data 硬切** 才稳 |
| **分帧挂主列表**（顶栏先、双 rAF/`listMounted` 后再挂 FlatList） | **硬禁**——见 §5 |
| 动画期父级 setCategories / setOrders 等 | 易带动 FlatList / Hero 父级 re-render |
| 动画结束再弹出本应与正文同在的控件 | 比掉帧更显眼；与正文同帧挂 |
| 过场 lite 正文 → 结束后自动换全文 | 双重 parse/layout，往往更卡 |
| 空盒 / 源文本冒充富文本撑布局 | 观感差；已否决 |
| 诊断 log 打满每一行 | log 本身制造 jank；封顶或采样 |
| 未确认就清 log | 须开发者满意后再 Phase C |
| 未问就上 FlashList/改导航模型等大方案 | 先穷尽本清单小改 |
| 把 MD 按行截断当成所有详情页默认 | 仅 §7 场景 |

---

## Canonical 参考

### Merchant 产品列表（列表进页）

实践中验证有效、且无「首屏控件延迟显现」的组合：

1. Home `pressIn` prefetch + 冷 `waitBrief` / 有 peek 立刻 navigate  
2. `peek` hydrate；bootstrap 命中 skipSet；有 peek 时 map/set 可放到 transitionEnd  
3. 分类条独立 memo 子树，`fetchEnabled` 等转场结束  
4. 推页期间 `data` 硬切 `ANIM_FIRST_ROWS`（如 10）  
5. 固定行高 + 主题样式缓存 + 操作图标 SVG tint 缓存  
6. **不做** 首屏图/徽章/操作的 lite 延后显示  

细节以当前 `ProductsListScreen` / `ProductListCard` 为准。

### Merchant / Customer 活动详情（详情进页 + 可选重 MD）

通用层：硬切订单/延后 below-fold、Hero↔列表 memo、指纹 skipSetDetail、`maxHeight`、推迟 measure、预览占位、Markdown `memo`、`runAfterScreenTransition`。  
MD 特化：收起按行截断 + 展开全文（§7）；以 `ActivityDescriptionCollapsible` / `ActivityDetailScreen`（merchant）与 customer `CatalogDetailScreens` 为准。

---

## 相关

| Skill / 规则 | 关系 |
|--------------|------|
| `list-to-detail-smooth-transition` | 数据路径 L0；本 skill 接渲染预算 |
| `ai-coding-principles` | 最小 diff；log 辅助诊断 Phase B/C |
| `expo-paginated-list-scroll` | 滚动恢复；与过场硬切正交 |
| `manual-npm-task-restart` | 勿代启 Metro / 各端 dev |
