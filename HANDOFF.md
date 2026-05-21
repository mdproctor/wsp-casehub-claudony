# Handover — 2026-05-21

**Head commit (project):** `ce55226` — docs(claude): update Qhorus test cleanup note for #119  
**Branch:** `main` — both repos. Epic `epic-gateway-reliability` **paused** (`.paused` on workspace main).

---

## Last Session

Completed and closed claudony#119 — `MeshResource` reactive refactor. Added `QhorusDashboardService` to Qhorus as the correct consumer integration tier (between entity services and MCP dispatch), simplified `MeshResource` to inject only it, fixed two CDI gotchas (`@Alternative @IfBuildProperty` interaction; `Panache.withTransaction()` named PU). PLATFORM.md boundary rule updated. Protocol `qhorus-consumer-integration-pattern.md` added. 475 tests, 0 failures.

## Immediate Next Step

The epic is paused. To resume: `work-resume` (switches both repos back to `epic-gateway-reliability`). Then `work-start` and pick up one of #100, #101, #102 below.

---

## What's Left

- **claudony#123** — test coverage for `/api/mesh/feed` and `/api/mesh/events` (filed this session) · S · Low
- **qhorus#175** — move `ChannelView`, `InstanceView`, `HumanMessageResult` DTOs to `casehub-qhorus-api` · M · Low
- **qhorus#176** — extract `toTimelineEntry`/`toChannelDetail` to shared `QhorusEntityMapper` · S · Low
- **qhorus#177** — align `Panache.withTransaction("qhorus", ...)` across all Qhorus reactive services · M · Low

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #100 | Channel panel catch-up on reconnect — poll `?after=lastId` on rejoin | M | Med | Core epic work |
| #101 | Re-register `ClaudonyChannelBackend` for all open-case channels on server restart | M | Med | Core epic work |
| #102 | Fleet-aware channel backend registration — push messages to all active nodes | L | High | Core epic work |
| #117 | `ClaudonyChannelBackend` — HumanParticipatingChannelBackend | M | Med | Fully unblocked (qhorus#131 + #153 closed) |

---

## Key references

- Blog: `blog/2026-05-21-mdp01-when-the-rule-is-right-but.md`
- Protocol (new): `docs/protocols/casehub/qhorus-consumer-integration-pattern.md` (parent repo)
- Garden: GE-20260521-0bd1e6, GE-20260521-d72294, GE-20260521-2b82e7, GE-20260521-49e7fd
- Test baseline: 4 core + 130 casehub + 341 app = 475 (2026-05-21)
- Qhorus tool count: 58 (8 Claudony + 50 Qhorus)
