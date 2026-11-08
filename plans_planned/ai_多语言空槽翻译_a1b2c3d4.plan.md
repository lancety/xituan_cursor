---
name: AI多语言空槽翻译
overview: 【planned / high】长文本多语言空槽 AI 填充：shouldRunFill 变更检测 + fillEmptyLocales + MULTILINGUAL_TEXT_TRANSLATE；保存前自动编排与语言切换批处理。块工具/插入产品描述已落地，本 plan 只含 AI 后续。
todos:
  - id: fill-empty-util
    content: codebase：fillEmptyLocales + pickSourceLocale + shouldRunFill（collapse 后源 hash）
    status: pending
  - id: ai-usecase
    content: backend：epAiUseCase.MULTILINGUAL_TEXT_TRANSLATE（payload 已 collapse；不查产品）
    status: pending
  - id: cms-save-hook
    content: CMS：保存前自动 fill（仅 shouldRunFill）；CRUD 不内嵌 AI
    status: pending
  - id: lang-switch-batch
    content: 商户支持语言变更批处理 worker（空槽 fill；弃用键删除）
    status: pending
isProject: false
---

# AI 多语言空槽翻译【planned / high】

## 前置（已完成，勿重做）

- 数据块框架：`aiMlAssistUtil` wrap / collapse / expand / `stripMarkersForDisplay`
- 首块 `product-desc` + CMS 团购「插入产品描述」
- 设计说明：[multilingual-ai-assist.md](../../xituan_agent/devGuide/ai/multilingual-ai-assist.md)
- Planned entry：`xituan_agent/devGuide/planned-work/entries/2026-11-multilingual-ai-empty-locale-fill.md`

## 范围

| 做 | 不做 |
|----|------|
| `shouldRunFill` + `fillEmptyLocales` + translate 端口 | 改 CRUD 内部调 DeepSeek |
| `MULTILINGUAL_TEXT_TRANSLATE` use case | 覆盖已填语言（除非显式强制重译） |
| CMS 保存前自动编排（过门闩） | 无变更检测的每次保存都烧 token |
| 语言 2→3 / 换语种 batch | 未注册 tag 的自动保护 |

## 变更检测（省 token）

`shouldRunFill` 本地零费用；任一失败则 **不调 AI**：

1. 无空语言槽  
2. 无非空源语言  
3. 源语言 **collapse 后** hash 未变且空槽集合未变  
4. 可选：本字段未 dirty  

通过后 `collapse → AI → expand`；成功更新 meta：`sourceLocale` / `sourceCollapsedHash` / `filledLocales` / `filledAt`（先会话内存，跨刷新 jsonb 另定）。

## 源语言优先级

显式 `preferredSource` → 商户内容主语言 → CMS UI 语言 → 商户启用语言序第一个非空 → `zh_cn` → `zh_tw` → `en`

硬约束：源非空；不按「最长文本」选源；块内文案靠 expand 按目标语替换，不靠 AI 译产品段。

## AI 用例契约

- `epAiUseCase.MULTILINGUAL_TEXT_TRANSLATE`（名可微调）
- Payload：`{ from, to, text }`（text 已 collapse）
- 约束：保留全部 `<!--xituan-` / `<!--/xituan-` 注释行；只译自由文案
- AI 服务 **不** resolve 产品 / 不读 offer

## 编排

```mermaid
flowchart LR
  Gate[shouldRunFill] -->|yes| Fill[fillEmptyLocales]
  Fill --> Collapse[collapseForAi]
  Collapse --> UC[MULTILINGUAL_TEXT_TRANSLATE]
  UC --> Expand[expandForLocale]
  Expand --> Submit[normal_CRUD]
```

- CMS：OfferForm（及通用 Multilingual）保存前  
- Batch：商户改支持语言；新语种空则 fill；弃用键删除  
