# Handover — 2026-05-22

**Head commit (project):** `92f7d26` — docs(claude): remove stale selected-alternatives note  
**Branch:** `main` — both repos. Clean close.

---

## Last Session

Closed #126: removed stale `quarkus.arc.selected-alternatives` block from both
`application.properties` files (22 bean entries, redundant since qhorus#172).
507 tests still passing. Discovered GitHub repo-move push redirect behaviour —
submitted as GE-20260522-b341ae. Updated CLAUDE.md to remove the now-false note.

## Immediate Next Step

Both repos on `main`. Pick next issue: #102 (fleet-aware backend registration)
or #117 (`HumanParticipatingChannelBackend`) — both recommended by previous handover.

---

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Next

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## Key references

- ADRs: `docs/adr/0006-channel-backend-registration-timing.md`, `0007-sse-channel-delivery-mechanism.md`
- Garden: GE-20260522-b341ae (GitHub repo-move push redirect)
- Test baseline: 4 core + 134 casehub + 369 app = 507 (2026-05-22)
- Blog: `blog/2026-05-22-mdp02-dead-config-live-redirects.md`
