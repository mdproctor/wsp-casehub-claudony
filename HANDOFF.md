# Handover — 2026-05-22

**Head commit (project):** `9b7dcae` — docs(design): apply design journal  
**Branch:** `main` — both repos. Epic `epic-gateway-reliability` **closed** (EPIC-CLOSED.md, deletion due 2026-06-05).

---

## Last Session

Completed and closed the `epic-gateway-reliability` epic: #100 (channel cursor catch-up), #98 (ClaudonyChannelBackend SSE delivery), #101 (restart re-registration). Also corrected blog routing — blogs now go directly to mdproctor.github.io, not staged in workspace. All 498 tests pass; DESIGN.md, ADRs 0006–0007, and protocols updated.

## Immediate Next Step

Both repos are on `main`. To start new work: open a new branch via `work-start`. Next candidates: #102 (fleet-aware backend registration) or #117 (`HumanParticipatingChannelBackend`).

---

## What's Left

- **claudony#123** — test coverage for `/api/mesh/feed` and `/api/mesh/events` · S · Low
- **claudony#124** — feed cursor `?after=<id>` support · S · Low  
- **claudony#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **claudony#128** — minor JS quality batch from #100 review · XS · Low
- **claudony#130** — EventSource error→fallback E2E test (Playwright route-abort) · S · Low
- **claudony#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **claudony#132** — `/api/mesh/events` double-frame bug fix · XS · Low
- **claudony#133** — minor quality batch from #101 review · S · Low
- **qhorus#175–177** — DTO/mapper/transaction cleanup from #119 · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Fleet-aware channel backend registration — push to all active nodes | L | High | Core epic follow-on; new branch |
| #117 | `HumanParticipatingChannelBackend` — bidirectional channel posting via backend | M | Med | Fully unblocked |
| #131 | ChannelEventBus-driven true push (replace 500ms tick in channelEvents) | M | Med | Investigation needed: cross-thread emit fix |

---

## Key references

- ADRs: `docs/adr/0006-channel-backend-registration-timing.md`, `0007-sse-channel-delivery-mechanism.md`
- Protocols: PP-20260522-c741d7 (claudony module boundary), PP-20260522-4c3d86 (idempotent registration)
- Garden: GE-20260522-1bc491 (SSE double-frame), GE-20260522-daca26 (emitter cross-thread), GE-20260522-2a4009 (onTermination concat), GE-20260522-567cc5 (InMemoryChannelStore UUID)
- Test baseline: 4 core + 130 casehub + 364 app = 498 (2026-05-22)
- Blog routing corrected: `blog → mdproctor.github.io` directly (no workspace staging)
