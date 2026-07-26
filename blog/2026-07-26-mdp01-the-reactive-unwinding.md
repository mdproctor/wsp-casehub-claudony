---
title: "The Reactive Unwinding"
date: 2026-07-26
entry_type: note
subtype: diary
status: draft
projects: [casehub-claudony]
tags: [reactive, virtual-threads, migration, spi]
---

The engine dropped all its `Reactive*` SPI interfaces. No deprecation period — the blocking variants exist, the reactive ones are gone. Claudony still implemented three of them, plus a compat shim that used reflection to handle `signal()` returning either `void` or `CompletionStage<Void>` depending on which engine SNAPSHOT you happened to have.

I'd expected this migration to be mechanical. Rename the classes, change `Uni<T>` to `T`, strip the reactive wrappers. And structurally it was — the blocking SPIs have identical semantics, just synchronous signatures. But the interesting part was what the reactive machinery had been hiding.

## What the reactive code was actually doing

The three SPI implementations — channel provider, worker provisioner, context provider — all had elaborate thread-safety choreography. The context provider maintained a "stable event loop context" created at `@PostConstruct`, used `emitOn` (not `runSubscriptionOn`, because that triggers Hibernate Reactive's internal VertxContext check), and detected whether it was running on the event loop thread to choose the right execution path. The channel provider had the same event-loop detection in `listChannels()` — returning empty from non-event-loop threads because channels wouldn't exist yet at that call site.

All of that goes away when you're on a virtual thread. The blocking SPI runs where the engine calls it, the Qhorus services handle their own transactions, and the thread-safety concerns about which context you're on simply don't exist.

## The ConcurrentHashMap trap

The one genuinely surprising moment was `ConcurrentHashMap.computeIfAbsent` throwing "Recursive update". The reactive channel provider used `Uni.memoize().indefinitely()` with `.onFailure().invoke(err -> cache.remove(key))` to evict failed entries from the layout cache. When I translated this to blocking, the natural equivalent puts the `cache.remove(key)` inside the `computeIfAbsent` lambda's catch block — which triggers the recursive modification guard.

The reactive pattern worked because `onFailure` runs asynchronously, after the map lock is released. The blocking translation looks equivalent but violates an invariant that was invisibly satisfied. The fix is simple — move the `try/catch` outside `computeIfAbsent` — but finding it required understanding why the reactive version ever worked at all.

## The Qhorus cascade

What I hadn't anticipated was that Qhorus also dropped its reactive services in the same SNAPSHOT window. `ReactiveChannelService`, `ReactiveMessageService`, `ReactiveCommitmentStore` — all gone, replaced by blocking `ChannelService`, `MessageService`, `CommitmentStore`. This turned `MeshResource` from a quick import rename into a full rewrite — every SSE endpoint that chained reactive dashboard calls needed restructuring.

The `CaseHubRuntimeCompat` reflection shim — the one that handled `signal()` returning either `void` or `CompletionStage<Void>` — is deleted entirely. The engine settled on `void`. One less class, one less reflection call, one less thing to wonder about when debugging.

## The shape of it

Thirty-four files changed, net reduction of about 300 lines. The reactive machinery — `runSubscriptionOn`, `emitOn`, stable event loop contexts, `Infrastructure.getDefaultWorkerPool()`, all the thread-safety comments explaining which context you're on and why — is replaced by direct method calls. The casehub module compiles and passes clean. The app module has some upstream SNAPSHOT drift in test compilation that's unrelated to this migration.

The `QhorusCausalLinkResolver` went from `@WithSession("qhorus")` with `Uni<Optional<UUID>>` to a plain blocking call against `MessageLedgerEntryRepository`. `SessionResource.getLineage()` lost its `.await().indefinitely()`. `CasehubResource.startAgent()` returns `Response` directly instead of `CompletionStage<Response>`.

Virtual threads make all of this possible without the performance trade-off that originally motivated the reactive design. The engine handlers call the blocking SPIs on virtual threads, the SPIs call Qhorus blocking services which manage their own transactions, and nobody needs to think about which event loop context they're on.
