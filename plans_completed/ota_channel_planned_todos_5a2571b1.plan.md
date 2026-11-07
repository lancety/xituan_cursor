---
name: OTA channel planned todos
overview: 暂不改 channel 代码；把自由 channel（方案 A）记入 planned-work 的「触发后才做」，并统一 entry 验收清单为两段结构。
todos:
  - id: readme-two-section
    content: README：Entry「期望目标结果」改为「计划要做的」+「触发后才做的」约定
    status: completed
  - id: entry-split-todos
    content: app-release-update-flow：拆清单；写入自由 channel 方案 A 触发项；钉死表注明暂不改
    status: completed
  - id: registry-next
    content: registry：更新 app-release-update-flow 下一动作与日期
    status: completed
isProject: false
---

# Planned-work：两段 todo + 推迟 OTA channel

## 决定（本轮只改文档，不改运行时代码）

- OTA channel **仍维持** `preview` | `production`（现状不动）。
- 方案 **A**（自由字符串；本地默认改名 `dev`；`dev-a`/`dev-b` 靠 env；第一次 `ota:prepare` 即出现；Platform 筛已有 channel；`production` 唯一生产名）写入 **触发后才做**，不排进当前窗口。
- 现有本地 `preview` 数据：触发实施时再决定 migrate→`dev` 或弃用旧行（备忘里写一句即可）。

## 文档改动

### 1. [planned-work README](xituan_agent/devGuide/planned-work/README.md)

在 **Entry 详文结构** 中把「期望目标结果」改为两段强制清单：

- **计划要做的** — 当前窗口内要推进/验收的项  
- **触发后才做的** — 每项写清 **触发条件**；未触发前不实施、巡检只确认条件是否出现  

同步改 `Last updated`。

### 2. [app-release-update-flow entry](xituan_agent/devGuide/planned-work/entries/2026-09-app-release-update-flow.md)

把现有「期望目标结果」拆开并刷新未勾项：

**计划要做的（当前窗口）**（示例，按现状勾选）

- 已完成项保留 `[x]`（migration、本地 Android OTA/强制更新等）
- `[ ]` 下一商店/本地包嵌入 code signing 公钥；本地 `ota:prepare:local` 签名验收（`code signing: yes` + 冷启动）
- `[ ]` production / iOS（TestFlight）链路验收（`ota:prepare:prod` + 生产 Platform 激活）
- `[ ]` 本 entry → `done`（上两项完成后）

**触发后才做的**

- `[ ]` **OTA 自由 channel（方案 A）**  
  - **触发**：多人并行本地 OTA 互相抢同一 `preview` active，或明确需要个人 channel（`dev-a` / `dev-b`）  
  - **内容**：放宽 enum/校验为 slug 字符串；local 默认 `preview`→`dev`；App/CLI 读 `EXPO_PUBLIC_OTA_CHANNEL` / `OTA_CHANNEL`；Platform 筛库中已出现 channel；保留 `production`；不建 channel 登记表  
  - 备注：改 channel 需重打原生包（写入 `updates.requestHeaders`）

可选：摘要/实施钉死表注明「channel 枚举暂不变；自由 channel 见触发清单」。

### 3. [registry.md](xituan_agent/devGuide/planned-work/registry.md)

`app-release-update-flow` 行：下一动作改为「签名验收 → TF/prod；自由 channel 见 entry 触发清单（勿提前做）」；更新日期。

## 刻意不做（本轮）

- 不改 codebase enum、backend 校验、两 App `app.config` / `ota-prepare`、Platform Select  
- 不跑 migration、不迁 `preview` 数据  
