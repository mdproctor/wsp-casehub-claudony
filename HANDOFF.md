# Handover — 2026-05-29

**Head commit (project):** `3624f7b` — adr: 0008 fleet-channel-backend-registration
**Branch:** `main` — #102 closed and merged

---

## Last Session

Completed #102: fleet-aware channel backend delivery. Core change: `ClaudonyChannelBackend` gains `@Observes ChannelInitialisedEvent` observer — the only registration path, replacing three explicit call sites. `createQhorusChannel()` calls `gateway.initChannel()` after channel creation (fires the event), then fires `CaseChannelCreatedEvent`. New `ChannelFleetBroadcaster` observes sync and calls `POST /api/internal/channels/sync` on each healthy peer. `ApiKeyAuthMechanism` fleet key now grants `fleet` role (not `user`), gating `ChannelSyncResource`. Pre-existing infra bugs fixed: stale `NoOpWorkloadProvider`, `WorkerDecisionEventCapture` CDI gap, engine#390 compat. 520 tests pass (4 + 137 + 379). ADR-0008 + 3 protocols captured.

## Immediate Next Step

`/work` — both repos on `main`, pause stack empty. Pick next from What's Left.

---

## What's Left

- **#118** — Fleet channel delivery (CLUSTER MessageObserver) · L · High — unblocked (was blocked on #102)
- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#94** — causedByEntryId causal chain · M · Med — blocked on engine#389
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **parent#81** — PLATFORM.md: ClaudonyChannelBackend uses SSE not WebSocket · XS · Low *(fix: it now fires `ChannelInitialisedEvent` — diagram may need update)*

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260529-baf565 (`@ObservesAsync` silently skipped when source uses `Event.fire()` — use `@Observes` + `Thread.ofVirtual()`)
- Protocols: PP-20260529-457e5f (channel backend via ChannelInitialisedEvent only), PP-20260529-68c422 (fleet clients `.register()` not `@RegisterProvider`), PP-20260529-e418f0 (fleet key = `fleet` role)
- ADR: `docs/adr/0008-fleet-channel-backend-registration-via-channel-initialised-event.md`
- Diary: `blog/2026-05-29-mdp02-one-way-to-register-a-backend.md`
- Test baseline: 4 core + 137 casehub + 379 app = 520 (2026-05-29, after #102)
