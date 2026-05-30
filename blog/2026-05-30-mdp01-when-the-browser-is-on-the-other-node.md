---
layout: post
title: "When the browser is on the other node"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [fleet, qhorus, messaging, sse]
---

The problem is simple to state: a worker posts a message on Node B. The browser watching that channel is connected to Node A. It gets nothing.

`ClaudonyChannelBackend` is LOCAL — it ticks the SSE fan-out on the node that received the message and nowhere else. In a single-node deployment that's fine. Once you have a fleet, it's a silent gap.

The fix I wanted was a CLUSTER-scoped `MessageObserver` — Qhorus already defines the SPI, `InProcessMessageBus` is the LOCAL default, but nobody had implemented the CLUSTER side. That was #118.

I brought Claude in to work through the design. The first question was whether the relay needed to carry the full message payload or just a channel-name tick. The answer depended on whether fleet nodes share a Qhorus database — if they do, a tick is enough, because the browser can fetch from its local Qhorus handle which resolves to the shared PostgreSQL. If they don't, you'd need to relay the full message content.

The deployment model settled it: production fleet uses PostgreSQL, H2 is single-node only. A tick is enough. `ChannelNotifyRequest` carries one field: `channelName`.

There's a correctness invariant that matters here. The fleet receiver endpoint has to call `ChannelEventBus.emit()` directly — never go through `MessageService.dispatch()`. If it did, Node A's `FleetMessageRelayObserver` would fire and relay back to Node B, which would relay to Node A, unbounded. `ChannelEventBus.emit()` is a pure in-process SSE tick with no observers — the loop can't form.

The spec went through multiple review rounds with Claude. The substantive ones: `recordSuccess()` was missing (the circuit breaker can't recover without it), the pre-commit race was documented incorrectly (it only applies to the blocking `MessageService` path — `ReactiveMessageService` fires observers post-commit, a fix already shipped as qhorus#193), and the data flow had observer and fanOut in the wrong order. Each of those would have surfaced as a bug or a misleading comment.

After implementation, the build fell apart.

The `mvn -U` I ran to clear cached 401 failures had pulled a newer Qhorus SNAPSHOT. The runtime jar had gained a new method on `ChannelBindingStore` — `findAll()`. The testing jar (`InMemoryChannelBindingStore`) was compiled against the old interface and didn't implement it. Everything compiled cleanly. The error only appeared at runtime: `AbstractMethodError: Receiver class InMemoryChannelBindingStore does not define or inherit an implementation of the resolved method 'abstract java.util.Map findAll()'`.

We rebuilt Qhorus locally from source. Both jars from the same source, interface consistency guaranteed.

There was a second failure that looked like our fault but wasn't. `ClaudonyLedgerEventCaptureTest.nullEventType_observerCompletesWithoutException` — the test fires a `CaseLifecycleEvent` with `eventType = null` and expects no exception. Our observer has a null guard. But a different engine bean, `CaseMemoryObserver`, observes the same event and calls `Set.of("CaseCompleted", "CaseCancelled", "CaseFailed").contains(event.eventType())`. `Set.of()` throws `NullPointerException` on `contains(null)` — unlike `HashSet.contains(null)`, which returns false. The exception propagated through CDI's async event infrastructure as a `CompletionException`, looking exactly like our observer had thrown.

The fix was one line: add `CaseMemoryObserver` to the `%test.quarkus.arc.exclude-types` list. Claudony has no `CaseMemoryStore` implementation — the observer is always a no-op here. It should have been excluded already.

The third failure was the timing one. `StatusAwareExpiryPolicyTest.expiresAtShellPromptWhenLastActiveIsOld` used `Thread.sleep(500)` to wait for a tmux pane command to change. Shell initialisation varies under load. Two lines of `Await.until()` replaced two lines of `Thread.sleep()`. The same test class already used `Await.until()` for the analogous case — it just hadn't been applied consistently.

525 passing, 0 failures.
