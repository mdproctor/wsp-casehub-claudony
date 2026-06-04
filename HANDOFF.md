# Handover — 2026-06-04

**Head commit (project):** `d088c8a` — docs(claude): update git workflow — repo transferred to casehubio/claudony, fork model gone
**Branch:** main (workspace + project)

---

## Last Session

Housekeeping: closed `issue-144-round-trip-cross-tenant-fix` (promoted missed spec `2026-05-15-casehub-startcase-roundtrip-design.md`, stamped branch), updated `origin` remote to `casehubio/claudony.git` (repo transferred; fork model gone), updated CLAUDE.md git workflow section. Garden entry submitted: GE-20260604-3aed8c (GitHub repo transfer silent redirect gotcha). Note: #142 (oversight `deniedTypes`) was completed in an unrecorded session — handover was stale for three days.

## Immediate Next Step

Start **Critical Path item 1**: WorkerExecutionManager tmux exit watcher. Nothing in `ClaudonyReactiveWorkerProvisioner` watches for tmux exit to fire `WorkflowExecutionCompleted`. Create an issue, then implement a virtual thread watcher in the provisioner.

---

## Critical Path to Real-World Examples

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Left

- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **parent#117** — PLATFORM.md + claudony.md: FleetMessageRelayObserver, test count 538, engine SNAPSHOT note · XS · Low
- **engine#231** — `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` (gates causedByEntryId) · S · Low

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

*Unchanged — `git show HEAD~1:HANDOFF.md`*
