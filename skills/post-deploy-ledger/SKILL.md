---
name: post-deploy-ledger
description: >-
  Pre-deploy and post-deploy phased rollout for changes that could break production
  (API/DB removals, multi-repo deploy order, WeChat review lag). Split into safe
  incremental Phase 1 plus deferred cleanup; register in post-deploy-ledger;
  track Gate and debt in registry.md. Use when designing features, migrations,
  API breaking changes, backend-before-client deploys, removing compatibility shims,
  or when the user says 检查 post-deploy / check post-deploy / post-deploy todo.
---

# Post-Deploy Ledger（分阶段部署与尾项追踪）

**Canonical devGuide:** [`xituan_agent/devGuide/post-deploy-ledger/README.md`](../../xituan_agent/devGuide/post-deploy-ledger/README.md)  
**巡检入口:** [`registry.md`](../../xituan_agent/devGuide/post-deploy-ledger/registry.md)  
**Global rule:** [`agent-global-dev-rules-checklist.md` §11](../../xituan_agent/devGuide/agent-global-dev-rules-checklist.md)

---

## Routine check: 「检查 post-deploy」

When the user says **检查 post-deploy**, **check post-deploy**, or asks whether any deferred cleanup remains:

1. Read [`registry.md`](../../xituan_agent/devGuide/post-deploy-ledger/registry.md) — **Active** table only.
2. If **zero** rows with status `active` or `blocked` (or `planned` with Phase 1 already deployed): report **无待收尾 post-deploy 债务**.
3. If **non-zero**: for each row, open the linked **entry** and summarize: ID, status, Gate, next action, unchecked post-deploy debt items.
4. If Gate appears satisfied, suggest scheduling Phase N cleanup PR — do not implement DROP/delete without user confirmation.

---

## When to apply (mandatory)

Apply this skill **before coding** when **any** of the following is true:

- Removing or renaming API fields/routes still used by **released** WeChat / Site / CMS builds
- Dropping DB columns or tightening constraints while old code may still read/write them
- Deploy order differs across repos (e.g. backend first, WeChat after app-store review)
- Replacing user-visible behaviour that old clients depend on (e.g. default address → last-used)
- Introducing a **temporary** compatibility layer (dual-write, deprecated fields, shim API)

If unsure, **default to phased plan + ledger entry** rather than a single big-bang deploy.

---

## Pre-deploy: design the change scope

### Step 1 — Assess blast radius

| Question | If yes → phased |
|----------|-----------------|
| Can every consumer ship the same day as backend? | No → phased |
| Will any **released** client break without the change? | Yes → Phase 1 must stay compatible |
| Is there a DB change old code cannot tolerate? | Yes → Phase 1 additive only |

Grep consumers across: `xituan_wechat_app`, `xituan_site`, `xituan_cms`, `xituan_platform`, external docs.

### Step 2 — Split phases

**Phase 1 (safe incremental — deploy first)**

- DB: **ADD** columns/tables only; **no DROP** of columns old code reads
- API: **keep** old fields/routes; add new fields; optional dual-write so old clients keep working
- Backend: new behaviour behind new fields or only when new clients send new params
- Frontend (if can ship same window): may use new fields; must not **require** backend cleanup

**Phase N (cleanup — deploy after Gate)**

- DROP columns, remove routes, stop dual-write, remove types/shims
- Only after **Gate** is met (see below)

Document both phases in devGuide / Cursor plan **and** create ledger entry **before Phase 1 merge**.

### Step 3 — Define Gate (must be verifiable)

Write Gate as checkboxes, not vague dates:

- Client: `wechat` version ≥ `x.y.z` **and** full rollout confirmed
- API: zero calls to deprecated route for N days (if measurable)
- Data: no reports/jobs depend on removed column

### Step 4 — Register ledger (before Phase 1 code lands)

1. Create `xituan_agent/devGuide/post-deploy-ledger/entries/YYYY-MM-<slug>.md` (use template in README)
2. Add row to `registry.md` (status `planned` → `active`/`blocked` after deploy)
3. Link from plan: `**部署尾项:** [entry](...)`

---

## Deploy Phase 1

Checklist:

- [ ] Migrations are **additive** only (next index under `xituan_backend/migrations/`)
- [ ] No removal of fields/routes old clients need
- [ ] Dual-write documented in entry if used
- [ ] After production deploy: **production smoke test** (old WeChat path if applicable)
- [ ] Update entry: deployed date, status `active` or `blocked`
- [ ] Update registry **Last updated** and **下一动作**

---

## Post-deploy: track until cleanup

**Human rhythm**

- Say **「检查 post-deploy」** (weekly / before prod release): open `registry.md` — any `active` or `blocked`?
- When Gate satisfied: schedule Phase N cleanup PR
- After cleanup deploy: mark entry `done`, move row to archived table in registry

**AI responsibility**

- When implementing a feature flagged as phased, **do not** implement Phase N cleanup in the same PR as Phase 1 unless user explicitly requests big-bang
- When user asks to "remove deprecated X", check registry first — Gate may not be met
- When closing a phased feature, update ledger entry checkboxes and registry

---

## Post-deploy cleanup (Phase N)

Before implementing:

- [ ] Re-read entry Gate — all items checked
- [ ] Grep repo for removed symbols (API path, column name, `isDefault`, etc.)
- [ ] New migration for DROP (separate file from Phase 1 ADD)
- [ ] Deploy backend cleanup only when consumer Gate satisfied (or accept documented risk)

After deploy:

- [ ] All **Post-deploy debt** items checked in entry
- [ ] Entry status → `done`; registry archived
- [ ] Remove duplicate bullets from repo `todo/` files

---

## Common patterns (Xituan)

| Scenario | Phase 1 | Phase N | Typical Gate |
|----------|---------|---------|--------------|
| Backend before WeChat review | Add column + dual-write old flag; keep old API | DROP column; delete API | WeChat version full rollout |
| API field rename | Return **both** old and new field | Remove old field | All consumers updated |
| Behaviour change | Backend supports both; old clients unchanged | Remove old path | Client metrics / version |
| Site + backend same day, WeChat lags | Phase 1 for WeChat compat | Phase 3 for cleanup | WeChat only |

Example entry: [`entries/2026-06-user-address-last-used-delivery.md`](../../xituan_agent/devGuide/post-deploy-ledger/entries/2026-06-user-address-last-used-delivery.md)

---

## Agent checklist (new feature / migration)

**Design**

- [ ] Assessed blast radius across wechat / site / cms / backend
- [ ] Phase 1 is additive and keeps old clients working
- [ ] Phase N cleanup listed with explicit Gate
- [ ] Ledger entry + registry row created (or updated if already exists)

**Phase 1 PR**

- [ ] No DROP / no delete route that released clients need
- [ ] Plan and entry linked

**After Phase 1 prod**

- [ ] Entry updated; registry status not left `planned`

**Phase N PR (only when Gate met)**

- [ ] Gate checklist complete in entry
- [ ] Registry archived after deploy

---

## Related

| Doc | Role |
|-----|------|
| [`devGuide/todo/README.md`](../../xituan_agent/devGuide/post-deploy-ledger/../todo/README.md) | Repo-local open work; link to ledger for phased debt |
| [`devguide-documentation`](../devguide-documentation/SKILL.md) | Where to put long-form design vs ledger |
| [`ai-coding-principles.mdc`](../../rules/ai-coding-principles.mdc) | Minimal diff + phased deployment section |
| [`migrations-stable-manual-only`](../../rules/migrations-stable-manual-only.mdc) | New SQL only in `migrations/` |
