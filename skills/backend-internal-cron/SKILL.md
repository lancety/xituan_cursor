---
name: backend-internal-cron
description: >-
  Xituan Backend unified internal cron: EventBridge (prod) or in-process HTTP
  self-call (dev/demo) → POST /api/internal/cron/:jobName with INTERNAL_CRON_API_KEY,
  single-flight DB lock, fast 202, domain services for work. Use when adding or
  migrating scheduled jobs, AusPost track poll ticks, ECS multi-task cron, or
  EventBridge API Destinations to Backend.
---

# Backend Internal Cron

权威方案：`xituan_agent/devGuide/backend-internal-cron-design.md`  
Track 业务算法：`xituan_agent/devGuide/delivery/auspost-mpb-track-poll-design.md`

## 何时用本 skill

- 新增 / 迁移 Backend 定时任务（含库存、首页缓存、metadata outbox、订阅、OpenIM、**auspost-track-tick**）
- 配置 EventBridge → Backend、或 dev/demo 进程内 cron
- 单飞锁、cron API Key、禁止多 ECS Task 各挂一份 node-cron

## 硬规则

1. **入口唯一**：`POST /api/internal/cron/:jobName` + 头 `x-cron-api-key`
2. **Env**：`INTERNAL_CRON_API_KEY`（+ 可选 `INTERNAL_CRON_API_KEY_PREVIOUS`）；与 `JOB_CALLBACK_SECRET` 分开
3. **快 ACK**：验 key → 抢锁 → **立刻 202/200**；长活后台跑（EB API Destination ~5s 超时）
4. **单飞**：同 `jobName` 未完成再触发 → skipped；**保底用数据库锁**（`pg_try_advisory_lock` 或 `cron_job_locks` 表）；Redis NX 仅可选加速
5. **业务在域模块**：cron 路由只分发，不堆业务
6. **prod**：EventBridge；**dev/demo**：`node-cron` HTTP 自调同一接口；自调 origin 用 `publicApiBaseUtil.resolveBackendPublicOrigin()`，禁止写死 127.0.0.1
7. **production** 禁止 `ENABLE_INPROCESS_CRON`

## 与 async jobs

| | Internal cron | Async jobs |
|--|---------------|------------|
| 触发 | 定时（EB / 本机 cron） | 业务 enqueue / SQS |
| 鉴权 | API Key | HMAC `JOB_CALLBACK_SECRET` |
| 路径 | `/api/internal/cron/*` | `/api/internal/jobs/*` |

压图/OCR 仍走 async jobs；定时 purge 可挂 cron job。

## 新增 job 检查清单

- [ ] 域内实现 `runXxx()`（可重入/幂等尽量）
- [ ] 注册 `jobName` → handler
- [ ] prod：EB Rule + API Destination + Connection API Key
- [ ] dev/demo：in-process 调度表增加频率
- [ ] 单飞锁 key = jobName
- [ ] 若为 Track：遵循 track-poll-design（Q=90、匀速、ETA 阶梯、游标/快照）；UI 订单详情不拉轨迹，点包裹/查看包裹 Modal 再读内部物流

## 相关 skill

- `auspost-mpb-api-service` — Track HTTP Core
- `redis-valkey-backend` — 可选锁加速，非保底
