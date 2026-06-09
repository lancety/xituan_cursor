---
name: xituan-codebase-change-scope
description: >-
  Decides which repos need codebase-related edits when changing shared
  submodules/xituan_codebase—map consumers by imports and API surface, update only
  apps that use new or changed exports, avoid treating every consumer submodule as
  in-scope by default. Use when adding or modifying shared types, enums, utils, API
  contracts, or any file under xituan_codebase and follow-up work in backend/cms/
  platform/site/wechat. For git pull/push order across all repos, use
  xituan-multirepo-codebase-sync instead; this skill does not replace that workflow.
---

# Xituan shared codebase: scoped change behavior (not full multirepo Git sync)

## What this skill is for

- **Planning and editing behavior**: which projects must change **because** they depend on new or altered shared code, versus projects that can stay untouched for this task.
- **After** you change `xituan_codebase`: which **consumer application repos** need import/call-site updates, type fixes, or config aligned with the new surface.

## What this skill is not

- **Not** the submodule pull/commit/push procedure across all five apps. For that, read and follow **`xituan-multirepo-codebase-sync`** when the user asks to push, align everything, or run the full chain.

## Single source of truth (avoid redundant edits)

- Shared code lives in the **`xituan_codebase` Git repo** (typically worked from `xituan_backend/submodules/xituan_codebase` first, per team convention).
- **Do not** paste the same logical patch into every consumer’s submodule working tree as a default habit. One commit on the codebase remote; other checkouts receive it via **git** when that project is updated (see multirepo sync skill).
- This skill’s “scope” means: **which consumer *main repos* and *app source*** need follow-up changes for the **feature or release slice**, not “touch all submodule folders every time.”

## Before changing shared code

1. **Name the public surface** you will add or change (exported symbols, DTO shapes, enum values, shared endpoints/constants).
2. **Find real consumers** in this monorepo (not assumed “all five”):
   - Search imports and paths, e.g. `@shared/`, `@xituan_codebase/`, `submodules/xituan_codebase`, re-exports from codebase into app `src/`.
   - Prefer **usage evidence** (import sites, API clients, serializers) over directory guesses.
3. **Classify the change**:
   - **Internal-only** (new private helper, no export change): often only the repo where you edit + tests; other apps unchanged unless they already imported that path.
   - **Additive** (new export, backward compatible): only repos that will **call or type against** the new API need updates in the same task.
   - **Breaking** (rename, remove export, semantic contract change): every repo that **currently** depends on the old surface needs a planned update; list them from grep results.

## After changing shared code

1. **Update application code** only in repos that **use** the changed surface (fix types, call sites, mocks, tests).
2. **Submodule pointer / version alignment** is a **release and Git** concern: only projects that **ship or CI-build** with the new codebase revision need a bump in that slice; unrelated apps are not obligated to submodule-bump in the same change just because codebase moved—unless the team policy or a breaking change forces it.
3. If unsure whether a project uses a symbol, **grep that project** for the module path or symbol name before editing it.

## Quick consumer map (heuristic, verify with search)

| Repo | Typical shared usage |
|------|----------------------|
| `xituan_backend` | Heavy: `@shared/*`, entities, API typing, validators |
| `xituan_cms` | Often: shared types, enums, display helpers under submodule paths |
| `xituan_platform` | When platform features import shared contracts or UI from codebase |
| `xituan_site` | When site imports `@xituan_codebase/*` or submodule paths |
| `xituan_wechat_app` | When mini-program shares types/utils from codebase |

Treat any row as **out of scope** until search shows an import or build dependency.

## Agent checklist (copy when relevant)

- [ ] Listed new/changed **exports** or contracts from codebase.
- [ ] Grepped **each candidate repo** for imports of those modules/symbols.
- [ ] Limited app/source edits to repos with **confirmed** usage (or documented “none”).
- [ ] Did not require unrelated repos to change “for symmetry.”
- [ ] If user later needs **full submodule alignment across all projects**, switched to **`xituan-multirepo-codebase-sync`** for Git steps.

## Handoff to multirepo sync

When the user wants **all** submodule checkouts and main repos pushed in the standard order, **stop relying on this skill alone** and apply **`xituan-multirepo-codebase-sync`**. This skill only narrows **who needs code or pointer follow-up** for a given feature.
