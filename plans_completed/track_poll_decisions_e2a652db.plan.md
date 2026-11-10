---
name: Track poll decisions
overview: 【completed 2026-11-10】统一 internal cron + Track 轮询已落地；EB 已配；待跑 migration + 发版。
todos:
  - id: confirm-partition
    content: 游标/事件表不用 HASH 50 分区（已拍板）
    status: completed
  - id: confirm-scheduler
    content: EB/本机→internal cron；全迁存量 cron；DB 单飞保底（已拍板）
    status: completed
  - id: sync-design-doc
    content: 同步 design + skill + planned-work（已完成）
    status: completed
  - id: implement-worker
    content: 落地 cron 框架+迁5个旧cron+Track DDL/tick/推送/读API/全端UI（EB prod 已配；migration/发版随部署）
    status: completed
isProject: false
---

# AusPost Track + Internal Cron — 定稿摘要

## 本轮确认

1. **第一期全做**：cron 框架 + 迁移现有 5 个 ECS cron + `auspost-track-tick`。
2. **单飞锁**：现网无现成 SET NX；`ICacheAdapter` 无锁；仅有阻塞式 `pg_advisory_xact_lock`。**保底用数据库**（`pg_try_advisory_lock` 或 `cron_job_locks` 表）；Redis 可选后加。
3. **文档已同步**（见下）。
4. **UI 按需拉物流**（与 Partner 无关；读内部快照）：订单详情**不**自动拉轨迹；商户「查看包裹」Modal 内每包裹下方展示，进 Modal/展开再请求；客户先包裹列表，**点击包裹**再拉该包裹物流。

## 权威文档

| 文档 | 路径 |
|------|------|
| 统一定时任务 | [backend-internal-cron-design.md](d:/projects/xituan_module/xituan_agent/devGuide/backend-internal-cron-design.md) |
| Track 算法 | [auspost-mpb-track-poll-design.md](d:/projects/xituan_module/xituan_agent/devGuide/delivery/auspost-mpb-track-poll-design.md) |
| Skill | [.cursor/skills/backend-internal-cron/SKILL.md](d:/projects/xituan_module/.cursor/skills/backend-internal-cron/SKILL.md) |
| planned | `backend-internal-cron`、`auspost-mpb-track-poll` |

## 实现未开始

代码 / CFN / DDL 等用户明确要求开工后再做。
