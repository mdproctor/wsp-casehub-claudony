# Handover — 2026-05-30

**Head commit (project):** `8d12d42` — fix: adapt to qhorus snapshot
**Branch:** `main` — #118 closed and merged

---

## Last Session

Completed #118: CLUSTER-scoped `MessageObserver` fleet tick relay. `FleetMessageRelayObserver` observes every Qhorus message dispatch and POSTs `ChannelNotifyRequest{channelName}` to each healthy fleet peer; peers call `ChannelEventBus.emit()` directly (no Qhorus re-dispatch — loop invariant). Requires shared PostgreSQL Qhorus for multi-node fleet. 525 tests pass (4 + 137 + 384). Also fixed two pre-existing build failures: `CaseMemoryObserver` CDI exclusion (`Set.of().contains(null)` NPE) and `StatusAwareExpiryPolicyTest` timing race (→ `Await.until()`). Post-close: adapted to new Qhorus snapshot (`ChannelInitialisedEvent.recovered` field, `createChannel` 13-param signature, `WorkerDecisionEventCapture` production exclusion).

## Immediate Next Step

`/work` — both repos on `main`, pause stack empty. Pick from What's Left.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#94** — causedByEntryId causal chain · M · Med — blocked on engine#389 (still open)
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **parent#117** — PLATFORM.md + claudony.md deep-dive: FleetMessageRelayObserver, test count 525 · XS · Low

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260530-1ce875 (`Set.of().contains(null)` throws NPE), GE-20260530-4359ee (`AbstractMethodError` from SNAPSHOT skew), GE-20260531-70e07c (`%test.quarkus.arc.exclude-types` ≠ production augmentation)
- Protocols: PP-20260530-ffa77a (tmux tests: `Await.until()` not `Thread.sleep()`), PP-20260530-9d18a0 (fleet notify endpoint must call `ChannelEventBus.emit()` directly)
- Diary: `blog/2026-05-30-mdp01-when-the-browser-is-on-the-other-node.md`
- Test baseline: 4 core + 137 casehub + 384 app = 525 (2026-05-30, after #118)
- Backup branch: `backup/pre-squash-main-20260530` (deletion: 2026-06-13)
