---
name: Self-hosted App OTA
overview: "【planned / high】基建已有：冷启动下载、下次冷启动生效、我的页长按强制 fetch+reload；Customer 有 ForceUpdateGate(minVersion)。缺口：每次打开检测热更新+最低版本+商店新版本；热更新/商店版各提示一次并记住 id/version；最低版本改为确认/取消 toast（非仅挡死门）。Merchant App 需接入与 Customer 同套检测。"
todos:
  - id: doc-update-entry
    content: 定稿 planned-work entry + registry
    status: completed
  - id: phase-force-update-base
    content: Platform minVersion + backend 查询 + Customer ForceUpdateGate（已有）
    status: completed
  - id: phase-ota-backend
    content: draft/presign/complete、manifest、content app-ota/（已有）
    status: completed
  - id: phase-ota-platform-cli
    content: CLI + Platform OTA 激活/回滚（已有）
    status: completed
  - id: phase-ota-apps-base
    content: expo-updates + 冷启动拉包 + 我的页长按更新（已有）
    status: completed
  - id: every-open-check
    content: "【high】每次 App 打开（含回前台策略自定）检测：热更新 / Platform minVersion / 商店可更新版本"
    status: pending
  - id: ota-prompt-once
    content: 热更新可用→提示一次；本地存已提示的 updateId，同 id 不再提示；用户确认后再 reload
    status: pending
  - id: min-version-confirm-cancel
    content: 低于 minVersion→确认/取消 toast（确认开商店；取消可继续用或按产品再定）；Merchant 也要挂检测
    status: pending
  - id: store-version-prompt-once
    content: 商店有新版本→类似热更新提示一次并记住 version，推荐下载
    status: pending
isProject: false
---

# 自建 OTA + 版本提示 UX（增量）

**2026-11-07 现状（已验收基建）**

| 能力 | 现状 |
|------|------|
| 冷启动 OTA | 下载；**下次**冷启动才应用（Expo 默认） |
| 手动 OTA | 「我的」版本号 **长按** → `fetchAndReload` |
| minVersion | Platform 可配；**Customer** `ForceUpdateGate` 挡死门；**Merchant App 未挂 Gate** |
| 商店新版本推荐 | **无** |
| 每次打开检测 + 一次性提示 | **无** |

## 新产品行为（planned-work high）

1. **每次打开**检测热更新、最低版本、商店新版本（回前台是否检测实施时定）。
2. **热更新**：有则提示一次；持久化当前 `updateId`，相同 id 不再提示；确认后下载/reload。
3. **最低版本**：提示后 **确认 / 取消** toast（不再只靠不可关挡板——若需强制仍可确认后跳商店）。
4. **商店新版本**：同热更新，提示一次并记住 version，推荐下载。

## 相关

- planned-work：`app-release-update-flow`（基建）+ 新 entry `app-update-check-ux`
- 自由 channel 仍见 entry「触发后才做」
