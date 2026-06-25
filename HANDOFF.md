# Handoff — 2026-06-25

**Head commit (project):** `700204b` — refactor(#157): migrate Worker imports to casehub-worker-api

## What landed this session

### claudony#157 — Worker import migration to casehub-worker-api

Migrated all Worker-related imports from `io.casehub.api.model` to `io.casehub.worker.api` across 8 files. Worker and Capability are now records (accessor changes: `getName()`→`name()`, constructor→factory). Lambda ambiguity on `Worker.Builder.function()` resolved with `WorkerFunction.Sync` wrapper. `casehub-worker-api` declared as direct dependency.

Also closed #162 (CI dispatch trigger) — already implemented, no code change needed.

Upstream dependency rebuild required: casehub-worker-api, casehub-engine-api, casehub-engine-common, casehub-engine-ledger, casehub-engine-testing, casehub-engine-persistence-memory, casehub-engine-scheduler-quartz, casehub-platform-identity (JwtVCValidator no-args constructor fix — committed to platform source, not yet pushed upstream).

## State

- main: `700204b`
- 557 tests pass locally (down from 587 — 30 tests migrated to engine-api in #159)
- #157 closed, #162 closed, branch stamped

## Upstream fix pending

`casehub-platform/identity` — added no-args constructor to `JwtVCValidator` to fix CDI proxyability error. Change is in local source only, not pushed to casehubio/platform. Must be committed and pushed in a platform session.

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #161 | Adopt casehub-pages for UI via Quinoa | L | High | Frontend architecture shift |
| #156 | Read agent provider config from casehub-ops | S | Med | Cross-repo dep |
