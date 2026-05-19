# Handover — 2026-05-19

**Head commit (project):** `0d90c54` — fix(sessions): add @Blocking to getLineage() and await CaseLineageQuery Uni
**Branch:** `epic-gateway-reliability` (both repos)

---

## What happened this session

**Bug fixed:** `SessionResource.getLineage()` was passing a raw `Uni<List<WorkerSummary>>` to
`Response.ok()` after the #115 reactive SPI migration — RESTEasy can't serialize a Uni, 500 at
runtime. Fix: `@Blocking` + `.await().indefinitely()`.

**Test baseline restored:** 475 passing, 0 failures (4 core + 130 casehub + 341 app).

**Qhorus#173** filed (getChannelTimeline() bypasses MessageStore abstraction) and fixed within the
session. Tool count updated 60→58 (8 Claudony + 50 Qhorus). `MeshResourceInterjectionTest` setup
rewritten to use `InMemoryChannelStore.put()` directly — `ReactiveChannelService.create()` now
wraps in `Panache.withTransaction()` requiring a Vert.x duplicated context unavailable on test thread.

**4 garden entries submitted:** Panache duplicated context, CDI delegate separation, RESTEasy Uni in
Response.ok(), reactive service blocking bypass.

---

## Immediate next

Run `work-start` and pick up #119 (MeshResource Uni<T> refactoring — replace `@Blocking` stopgap
with proper reactive return types). Self-contained, no upstream deps.

---

## Open issues

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Blog: `blog/2026-05-19-mdp02-debugging-test-baseline.md`
- Garden: GE-20260519-4a42e6, GE-20260519-f9624b, GE-20260519-d32fc0, GE-20260519-f33c66
- Journal: `design/JOURNAL.md` (workspace)
