# Handover — 2026-05-28

**Head commit (project):** `cc314d0` — docs(design): apply journal — ChannelEventBus push pipeline replaces 500ms tick (#131)
**Branch:** `main` — #131 closed and merged

---

## Last Session

Completed #131: replaced 500ms SSE polling tick in `MeshResource.channelEvents()` with a
merged-signals push pipeline — `ChannelEventBus` push events (`emitOn(workerPool)` to fix
cross-context Vert.x frame drops) and a 30s preference-driven heartbeat emitting `"[]"`.
Single `transformToUniAndConcatenate` across the merged stream prevents concurrent
`getTimeline()` calls and duplicate frames. `ChannelHeartbeatInterval` preference key added.
`MeshResource` now `@ApplicationScoped`. 520 tests pass.

## Immediate Next Step

Pick up next issue from What's Left. Both repos on `main`, clean state.
Start with `/work` — pause stack should be empty after work-end.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#102** — Fleet-aware channel backend registration (all nodes) · M · Med — #98 now confirmed clean
- **#116** — Reactive Qhorus stack for PostgreSQL · M · Low — both qhorus blockers (qhorus#141, qhorus#161) closed
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med
- **parent#81** — PLATFORM.md: ClaudonyChannelBackend uses SSE not WebSocket · XS · Low

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260522-daca26 (revised — Vert.x cross-context SSE, both solutions documented), GE-20260527-8c3ff5 (concurrent transformToUniAndConcatenate → duplicate frames)
- Protocol: PP-20260527-d95fed (claudony SSE timing params → PreferenceKey, not ClaudonyConfig)
- qhorus#204 filed: gateway human_observer deduplication (tracked for removing channelRegistrationLocks sync)
- Test baseline: 4 core + 135 casehub + 381 app = 520 (2026-05-28, after #131)
