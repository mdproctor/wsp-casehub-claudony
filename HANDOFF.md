# Handover — 2026-05-25

**Head commit (project):** `03e4b9e` — docs: sync test baseline to 510 (4+135+371) after #114, #120, #127
**Branch:** `main` — both repos. CI green.

---

## Last Session

Batch of S/XS issues. Fixed openChannel concurrent init race (#120) via
`computeIfAbsent` + `Uni.memoize().indefinitely()` with failure eviction.
Fixed channel cursor accumulation on worker switch (#127). Test style cleanup
(#114). Closed #112 as already resolved. #129 blocked on qhorus#201.

## Immediate Next Step

Both repos on `main`, CI green. Pick up any item from What's Next — all unblocked.

---

## Closed Branches (all work incorporated)

*Unchanged — `git show HEAD~1:HANDOFF.md`*

Also closed this session:
- `issue-batch-xs-s-cleanup` — merged, stamped closed, deletion due 2026-06-08

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#129** — Replace ChannelView/InstanceView with canonical API types · S · Low · blocked on qhorus#201
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med
- **qhorus#201** — QhorusDashboardService return canonical API types · S · Low · unblocks #129

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260524-c1e573 (Quarkus exclude-types silently fails for extension beans)
- Blog: `2026-05-24-mdp01-dependency-that-didnt-exist.md`
- Test baseline: 4 core + 135 casehub + 371 app = 510 (2026-05-25)
