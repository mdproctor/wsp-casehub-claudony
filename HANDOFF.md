# Handover — 2026-05-29

**Head commit (project):** `692e6c7` — docs(#116): sync CLAUDE.md
**Branch:** `main` — #116 closed and merged

---

## Last Session

Completed #116: `listChannels()` swapped from `listAll()` + client-side filter to `findByNamePrefix()` (server-side). PostgreSQL reactive IT added — 4 tests against a real `postgres:17-alpine` container via `QuarkusTestResource` (required because `getConfigProfile()` replaces `%test` entirely, making production H2 URL active; Dev Services can't override it). `@RunOnVertxContext + UniAsserter` required for test methods calling Panache directly. Engine package refactoring (`internal.*` → `common.*`) fixed as chore. #94 (causedByEntryId) filed as engine#389 — blocked on engine firing `WorkerStarted` lifecycle event. 520 tests pass.

## Immediate Next Step

`/work` — pause stack empty, both repos on `main`. Pick next from What's Left.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#102** — Fleet-aware channel backend registration (all nodes) · M · Med
- **#118** — Fleet channel delivery (CLUSTER MessageObserver) · L · High — blocked on #102
- **#94** — causedByEntryId causal chain · M · Med — blocked on engine#389
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med
- **parent#81** — PLATFORM.md: ClaudonyChannelBackend uses SSE not WebSocket · XS · Low

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260529-18fc5f (QuarkusTestProfile.getConfigProfile() replaces %test — Dev Services config priority), GE-20260529-78ecb9 (@RunOnVertxContext + UniAsserter for Panache @QuarkusTest)
- Protocol: PP-20260528-ac6d93 (reactive-pg Dev Services named-datasource profile)
- engine#389 filed: WorkerStarted lifecycle event + causedByEntryId for causal chain (#94)
- Test baseline: 4 core + 135 casehub + 381 app = 520 (2026-05-29, after #116)
