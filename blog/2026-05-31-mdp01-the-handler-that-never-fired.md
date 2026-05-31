---
layout: post
title: "The handler that never fired"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [sse, quarkus, causality, ledger]
---

A listener that registers without complaint, data flowing from server to browser, an open connection — and yet the panel never updates. That's how `SseMeshStrategy` had been sitting in the codebase.

We found it while working on `#125`, the SSE reconnect cursor for `/api/mesh/events`. When we looked at the existing handler to wire in the cursor logic, it was `addEventListener('mesh-update', ...)` — listening for a named SSE event that the server never sent. RESTEasy Reactive wraps `Multi<String>` items as bare `data:` frames when producing `text/event-stream`. No `event: mesh-update` line. So the listener waited patiently for something that could never arrive.

It was only safe because `claudony.mesh.refresh-strategy=poll` is the default. The SSE path existed; it just didn't do anything. Once we understood the RESTEasy behaviour — `Multi<String>` gives you `data:` only; `OutboundSseEvent` is what you need if you want `event:` or `id:` fields — the fix was straightforward. Change the handler to `source.onmessage`, track `_eventId` from the payload, reconnect with `?after=<lastEventId>` on error.

The cursor logic itself is simpler than the SSE spec makes it sound. The server embeds `_eventId` (the max feed message ID) in every JSON frame. On reconnect, the client opens `EventSource('/api/mesh/events?after=42')`. The server checks: first tick, `after >= 0`, current max feed ID ≤ after — skip. Wait for the next tick with actual new state. No `Last-Event-ID` header, no SSE `id:` fields, no `OutboundSseEvent` refactor. Application-level cursor piggybacking on the existing JSON payload. Adding content equality before re-rendering handles the rest: if channels, instances, and feed are structurally identical to what was already displayed, skip the DOM work entirely. The flicker on reconnect disappears.

---

The other half of the branch was `#140` — wiring `causedByEntryId` into the provisioner and ledger capture. The design was established at the engine level months ago: when a worker is provisioned, the provisioner should return the ledger entry ID of the Qhorus COMMAND that triggered it; the engine fires a `WorkerStarted` lifecycle event; the ledger capture observes it and sets `entry.causedByEntryId`. What was missing was Claudony's end of the wire.

The tricky part is that `CaseLifecycleEvent` doesn't carry `causedByEntryId` — by design, shared events don't carry consumer-specific fields. So there's a `ConcurrentHashMap<UUID, UUID>` bridge: the provisioner stores `(caseId → resolvedEntryId)` before returning, and `ClaudonyLedgerEventCapture` drains it when `WorkerStarted` fires. The bridge is safe because `fireAsync()` runs after the provisioner's Uni resolves — the store always happens before the drain.

In practice, the map will be empty until engine#231 threads `triggerChannelId` and `triggerCorrelationId` through `ProvisionContext`. I know that going in. The scaffold is correct — the data flow will complete as soon as the engine ships its side. For now, `causedByEntryId` is null in every new ledger entry, same as before.

---

The session also handed us an unexpected adaption job. Midway through the work, the engine SNAPSHOT updated `CaseLifecycleEvent` from a 7-field to an 8-field record — a `tenancyId` added as the second component. The ledger entity followed suit: `CaseLedgerEntry.tenancyId` became non-null at the Hibernate constraint level. A cascade of compile errors across every test that constructed the event or wrote a ledger entry directly.

The engine was in a half-updated state — `scheduler-quartz` and `persistence-memory` production code still called the old constructor. We fixed the engine handler files to unblock the test suite, defaulted `tenancyId` to `"default"` in `ClaudonyLedgerEventCapture`, and left a note that the engine round-trip test will stay red until the engine finishes its own changes and publishes a consistent SNAPSHOT. Not ideal to leave a test broken, but the alternative was sitting on it for however long that takes.

531 of 532 tests pass. The one that doesn't is the engine's problem to finish.
