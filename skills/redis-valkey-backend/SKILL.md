---
name: redis-valkey-backend
description: >-
  Xituan backend Redis/Valkey usage—ElastiCache Serverless Valkey to node migration,
  ioredis single client, ICacheAdapter, rate-limit middleware, key naming, env vars,
  and Valkey vs Redis compatibility. Use when implementing Redis cache, API rate limiting,
  ElastiCache wiring, REDIS_URL, ioredis, Valkey, or migrating from in-memory Map
  to shared Redis across ECS tasks.
---

# Redis / Valkey backend development

Read **`xituan_agent/devGuide/redis_search/redis/valkey-development.md`** for full detail. This skill is the agent checklist for code changes.

## Infra roadmap (do not fight it in code)

1. **Serverless Valkey** — rate limit + short TTL cache (now)
2. **Valkey node** — scale / predictable cost (env + pool only)
3. **Redis node** — only if Redis Stack / modules required (isolated domain code)

**Never** branch business logic on Serverless vs node. **Never** scatter `new Redis()` outside `shared/redis/`.

## Before writing code

- [ ] Confirm use case: rate limit, TTL cache, lock, or Redis-only module?
- [ ] If Redis-only module (Search/JSON) → separate domain folder; confirm Valkey insufficient
- [ ] Check existing `ICacheAdapter` and `getDefaultCache()` in `xituan_backend/src/shared/cache/`
- [ ] WAF already limits edge; Redis limits **per user/merchant** and **cross-ECS** (complement WAF)

## Required architecture

| Piece | Path (create if missing) |
|-------|--------------------------|
| Connection singleton | `src/shared/redis/redis-client.util.ts` |
| Rate limit helpers | `src/shared/redis/rate-limit.redis.util.ts` |
| Express middleware | `src/shared/middleware/rate-limit.middleware.ts` |
| Redis cache | `src/shared/cache/redis-cache.adapter.ts` |

Use **`ioredis`**. Env: `REDIS_URL` (`rediss://` in prod). Optional `REDIS_KEY_PREFIX`.

### Key prefix

`xituan:{env}:{purpose}:{scope}:{id}` — e.g. `xituan:prod:rl:ip:{hash}`

### Local / no Redis

If `REDIS_URL` unset: keep **in-memory** cache; skip Redis rate limit (fail open + log). Do not require Redis for `npm run dev`.

## Implementation rules

1. **Util export:** wrap helpers in `redisClientUtil`, `rateLimitRedisUtil` — files `*.redis.util.ts` or `redis-client.util.ts` per project convention
2. **No `any`** in TypeScript
3. **Comments in English**
4. Domains call **`getDefaultCache()`** or **`rateLimitRedisUtil`** — not raw Redis commands
5. Rate limit: **429** + `Retry-After`; align sensitive paths with WAF `RateLimit-Sensitive`
6. Replace process-local `Map` rate limits (e.g. password-reset) when adding shared Redis util
7. Do not store secrets or large payloads in Redis without explicit review

## Valkey-safe commands (default)

`GET`, `SET`, `SETEX`, `DEL`, `INCR`, `EXPIRE`, `TTL`, `SET NX EX`, pipelines.

Avoid depending on **RediSearch / RedisJSON / Redis Stack** unless ticket explicitly targets Redis node migration.

## Switching infra (agent should not refactor domains)

| Change | Code impact |
|--------|-------------|
| Serverless → Valkey node | Update `REDIS_URL`, pool tuning |
| Valkey → larger / replica | AWS only; optional read URL for cache reads |
| Valkey → Redis node | `REDIS_URL` + module-specific code only |

## Related docs

- `xituan_agent/devGuide/redis_search/redis/valkey-development.md` — full guide
- `xituan_agent/devGuide/backend-protection-layers-and-scale-notes.md` — WAF, PgBouncer, layer-5 DB ticket backlog
- `xituan_backend/todo/metadata-schema-etag-redis-l2.md` — metadata Redis L2 cache

## Suggested ticket order

REDIS-1 client → REDIS-2 `RedisCacheAdapter` → REDIS-3 middleware → REDIS-4 migrate password-reset Map → REDIS-5 hot-path cache
