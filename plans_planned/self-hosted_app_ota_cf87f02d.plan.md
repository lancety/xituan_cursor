---
name: Self-hosted App OTA
overview: "【in_progress / high】代码已落地：硬/soft 按 OS；OTA reload；底部条按钮；4h；待真机验收。"
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
  - id: toast-actions
    content: 两端 AppToast Viewport 支持确认/取消（及可选次要官网按钮）；更新场景强制 Viewport
    status: completed
  - id: dual-tier-settings
    content: Platform 拆 min/recommended 按 OS；apkUrlAndroid；recommended≥min；旧 min 双写
    status: completed
  - id: every-open-check
    content: 冷启动必查；回前台 4h；OTA / 硬 min / soft
    status: completed
  - id: ota-prompt-once
    content: OTA 提示一次；确认已下载则 reload，否则 fetch+reload；取消记 updateId
    status: completed
  - id: hard-min-merchant
    content: 硬 Gate 按 OS；Merchant 挂根
    status: completed
  - id: soft-store-prompt
    content: soft 仅 current<recommended；主商店+次要官网；取消记 version
    status: completed
  - id: prompt-priority
    content: 同屏只一个：硬 min → OTA → soft
    status: completed
  - id: device-qa
    content: 真机/发版验收
    status: pending
isProject: false
---

# 自建 OTA + 版本提示 UX（增量）

## 已拍板（2026-11-07，含第二轮确认）

1. soft/OTA UI：底部条 **带确认/取消**（现 toast 无按钮 → **扩 Viewport**；不用 ToastAndroid）。
2. OTA 确认：已静默下载 → **只 reload**；否则 fetch+reload。
3. soft：仅 `current < recommended`；空或 `recommended <= current` 不提示。
4. Platform：`recommended >= min`（同平台）。
5. 旧 `minVersion` → iOS/Android 同值。
6. Android CTA：双 URL → **主开商店 + 常驻「官网下载」**；`openURL(商店)` catch 再开 apk（单靠失败回调不可靠）。
7. Customer + Merchant 同波。
8. 回前台含硬 min：**4h** 节流（暂定）。

官网 APK 可信实践 → planned-work **mid** `android-apk-sidecar-trust`。

## 相关

- `app-release-update-flow`（done）+ `app-update-check-ux` + `android-apk-sidecar-trust`
