# Handover — 2026-05-24

**Head commit (project):** `4711555` — fix(deps): rename casehub-testing to casehub-engine-testing — fixes CI
**Branch:** `main` — both repos. CI green.

---

## Last Session

Fixed CI: `casehub-testing` (parent repo module, never published) renamed to
`casehub-engine-testing` (identical jar, published by engine CI). Also resolved
local test failure from casehub-ledger Merkle frontier beans — coordinated
`@DefaultBean` no-op fix with ledger session. CI green, 508 tests passing.

## Immediate Next Step

Both repos on `main`, CI green. Pick up any item from What's Next — all unblocked.

---

## Closed Branches (all work incorporated — deletion due 2026-06-07)

**Project repo:**
- `epic-gateway-reliability` — merged
- `feat/rename-to-claudony` — diverged but rename complete (no remotecc refs remain)
- `issue-122-commitment-quality-batch` — merged
- `issue-126-remove-selected-alternatives` — merged
- `issue-136-test-cleanup` — merged
- `issue-138-qhorus-184-cleanup` — squashed to main; backup/pre-squash-main-20260523 preserves pre-squash state
- `issue-6-sla-propagation` — merged
- 9x `worktree-agent-*` — all work incorporated
- `backup/pre-squash-main-20260508` — past 14-day retention
- `backup/pre-squash-v1-main-20260508` — past 14-day retention

**Workspace:** `epic-gateway-reliability`, `issue-122-*`, `issue-126-*`, `issue-136-*`, `issue-138-*` — mirrors of project branches, all merged.

**Keep:** `backup/pre-squash-main-20260523` — 2 days old, deletion due 2026-06-06.

---

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260524-c1e573 (Quarkus exclude-types silently fails for extension beans)
- Blog: `2026-05-24-mdp01-dependency-that-didnt-exist.md`
- Test baseline: 4 core + 133 casehub + 371 app = 508 (2026-05-24)
