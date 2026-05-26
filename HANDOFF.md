# Handover — 2026-05-26

**Head commit (project):** `b1af048` — chore(#139): use CaseChannel.channelName() — engine-api owns case channel naming format
**Branch:** `main` — XS/S batch closed and merged

---

## Last Session

Completed the XS/S batch (`issue-113-139-xs-batch`): #113 adapted tests to engine#349 changes (CaseLifecycleEvent traceId, QhorusMessageSignalBridge exclusion, NoOpJobScheduler @DefaultBean); #139 replaced local `CHANNEL_PREFIX = "case-"` with `CaseChannel.channelName()` / `CaseChannel.CASE_CHANNEL_PREFIX`. 3 commits squashed to 2, pushed to casehubio/claudony/main. #122 and #135 were already closed. 510 tests pass.

## Immediate Next Step

Pick up the next open item — the only remaining XS/S is gone; next is **#125** (SSE Last-Event-ID reconnect, M · Med) or **#131** (ChannelEventBus true push, M · Med), both in What's Left. Open branches: `epic-gateway-reliability`, `issue-126-remove-selected-alternatives`, `issue-136-test-cleanup`, `issue-138-qhorus-184-cleanup`, `issue-batch-xs-s-cleanup` (32h ago) — check pause stack before starting new work.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260526-34a4c4 (scheduler-quartz CDI cascade), GE-20260526-2ee43b (engine bean SmokeTest failure)
- Test baseline: 4 core + 135 casehub + 371 app = 510 (2026-05-26)
