---
layout: post
title: "Phase 15: Auditing the Audit Trail"
date: 2026-04-29
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub, ledger, concurrency, debugging, testing]
excerpt: "An external audit finds two production bugs in ClaudonyLedgerEventCapture: a swallowed persistence exception that silently drops ledger entries, and a MAX() sequence query with no UNIQUE constraint that allows duplicate sequence numbers under concurrent writes."
---

The inevitable question when you're building in this space: there are dedicated CLI wrapper UIs already, so why does Claudony exist? My answer is that Claudony isn't a terminal emulator. The terminal is one panel out of three. The value is watching multiple Claude workers coordinate on a shared case, seeing their Qhorus channel conversations in real time, and injecting a human directive that gets recorded in the lineage. A CLI wrapper doesn't do any of that because it isn't built on CaseHub and Qhorus — it's just plumbing.

And even at minimum viable — even if the three-panel dashboard stays exactly as it is — the code is a working reference implementation of how the three components wire together. A running reference is more useful than a diagram.

The rest of this session was debugging. An external audit of `ClaudonyLedgerEventCapture` — Claudony's replacement for the casehub-ledger event capture bean — turned up two production issues.

## Silence where there should be noise

The first bug: a `catch(Exception e)` block was wrapping `em.persist()`. Any persistence failure — database unavailable, constraint violated, whatever — was being logged and swallowed. The CDI observer completed successfully. The `CompletionStage` returned by `fireAsync()` resolved. The caller saw success. The ledger entry didn't exist.

This is the classic async observer trap. Quarkus fires `@ObservesAsync` on a managed executor thread in its own `@Transactional` context. If the observer throws, the `CompletionStage` completes exceptionally — but only if the throw reaches the CDI infrastructure. A try/catch kills that signal. `casehub-engine`'s equivalent never had one. The fix was to remove the catch block.

## Sequence numbers that don't

The second bug is subtler. `nextSequenceNumber()` was using `MAX(sequenceNumber) + 1`. Under concurrent writes — two observers firing for the same `caseId` — both threads can read the same `MAX` before either commits. Both derive the same next value. Both insert successfully. The ledger silently stores two entries with `sequenceNumber = 1`.

The schema is why it's silent. `ledger_entry` has an index named `idx_ledger_entry_subject_seq` on `(subject_id, sequence_number)`. The name implies uniqueness. It's a plain performance index. There's no `UNIQUE` constraint on that column pair anywhere in the migration — two rows with the same subject and sequence insert without complaint.

The fix matches `casehub-engine`: `ORDER BY sequenceNumber DESC` with `setMaxResults(1)` and `findFirst()`, mapped to `sequenceNumber + 1` or `1` if empty. Same theoretical race window, but it uses the index and matches the authoritative pattern:

```java
return em.createQuery(
        "SELECT e FROM CaseLedgerEntry e WHERE e.subjectId = :caseId ORDER BY e.sequenceNumber DESC",
        CaseLedgerEntry.class)
    .setParameter("caseId", caseId)
    .setMaxResults(1)
    .getResultStream()
    .findFirst()
    .map(e -> e.sequenceNumber + 1)
    .orElse(1);
```

We wrote the tests first — happy path field verification, sequence increment, sequence independence across cases, null guards. The pattern that works for testing `@ObservesAsync` observers: `fireAsync().toCompletableFuture().join()` blocks until the observer has committed its own transaction, so reads immediately after are reliable without any polling.
