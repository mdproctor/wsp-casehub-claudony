# Handover — 2026-05-22

**Head commit (project):** `fedf22d` — docs(claude): update test baseline to 507 after quality batch  
**Branch:** `main` — both repos. Clean close, no open branches.

---

## Last Session

Implemented and closed the `issue-122-commitment-quality-batch` branch: #122 (correlationId→Commitment wire-up), #123 (feed tests), #128 (JS quality), #130 (EventSource E2E), #132 (SSE double-frame fix — global mesh dashboard was silently delivering nothing since #101), #133 (ChannelEventBus TOCTOU, registration lock, logging). Test baseline 498→507. #124 (feed cursor) deferred — Qhorus frozen.

## Immediate Next Step

Both repos on `main`. Start new work: `work-start` then pick #102 (fleet-aware backend registration) or #117 (`HumanParticipatingChannelBackend`).

---

## What's Left

- **#124** — feed cursor `?after=<id>` support — deferred, needs Qhorus `getFeed()` change · S · Low
- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **#135** — Add `correlationId` param to `postToChannel()` SPI in casehub-engine (removes content-coupling parse from #122) · S · Med · cross-repo
- **#136** — Test comment cleanup in `meshEvents_sseFrameIsValidJson` · XS · Low
- **#137** — Strengthen multi-channel feed test to assert both channel names · XS · Low
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- ADRs: `docs/adr/0006-channel-backend-registration-timing.md`, `0007-sse-channel-delivery-mechanism.md`
- Protocols: PP-20260513-7c227e, PP-20260522-c741d7, PP-20260522-4c3d86
- Garden: GE-20260522-39837c (SSE double-frame), GE-20260522-98b286 (ConcurrentHashMap TOCTOU), GE-20260522-6c22a3 (computeIfPresent-null)
- Test baseline: 4 core + 134 casehub + 369 app = 507 (2026-05-22)
- Blog: `blog/2026-05-22-mdp01-what-the-sse-stream-was-hiding.md`
