---
layout: post
title: "One way to register a backend"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [qhorus, fleet, cdi, channel-backend]
---

I started this branch expecting a small fleet plumbing task and found out immediately that the plumbing was already broken — even on a single node.

The bug was `createQhorusChannel()`. It called `channelService.create()` to persist the channel in the Qhorus database but never called `gateway.initChannel()`. Without `initChannel()`, the `ChannelGateway` registry doesn't know the channel exists. `fanOut()` reads from that registry, finds nothing, and returns silently. `ClaudonyChannelBackend.post()` is never called. The SSE panel still worked — it polls the database directly every 500ms — so the missing backend was invisible. But invisible is exactly the problem when you're trying to build fleet delivery on top of it.

I wrote a diagnostic test to prove this. First attempt failed for the wrong reason: the test dispatched a message through `QhorusDashboardService.sendHumanMessage()`, which routes through `ReactiveMessageService.dispatch()`, which turns out not to call `fanOut()` at all. That's a separate Qhorus issue (#193, deferred). The test was testing the wrong path. We rewrote it to call `gateway.fanOut()` directly, which tested the actual delivery chain — and confirmed the bug.

The fix I'd planned was to wire each registration call site to also call `initChannel()`. But there were three of them — startup bootstrap, SSE panel open, channel creation — and none of them propagated to fleet peers. Adding fleet fan-out to each site would mean three places to maintain, and any new registration site would silently lack fleet propagation. The right answer was to collapse them.

Qhorus has the mechanism already: `ChannelInitialisedEvent` fires whenever `initChannel()` is called — at startup (for all persisted channels) and on demand. External backends are meant to observe this event to register themselves. We weren't using it. So we did: `ClaudonyChannelBackend` now observes `ChannelInitialisedEvent` and registers for any `case-*` channel. `createQhorusChannel()` calls `initChannel()` after creating the channel, which fires the event, which triggers registration. On startup, `ChannelGateway.onStart()` calls `initChannel()` for every persisted channel, which fires the event, which handles the startup case automatically. `bootstrapChannelBackends()` — the manual startup method that did this explicitly — became dead code and was removed.

The fleet side is a new `ChannelFleetBroadcaster`. When a channel is created, `createQhorusChannel()` fires a `CaseChannelCreatedEvent` CDI event after calling `initChannel()`. The broadcaster observes this and calls `POST /api/internal/channels/sync` on each healthy peer, which triggers `initChannel()` on the peer's gateway, which fires `ChannelInitialisedEvent` there, which registers the backend.

One thing Claude caught during the broadcaster implementation: I'd specced `@ObservesAsync` for the observer, assuming it would deliver the event on a background thread regardless of how it was fired. It wouldn't. CDI 4 is unambiguous: `Event.fire()` only notifies `@Observes` (synchronous) observers. `@ObservesAsync` observers only receive events fired with `Event.fireAsync()`. Since `createQhorusChannel()` runs inside a reactive `Uni.map()` callback and uses `Event.fire()`, async observers get nothing — no error, no warning, just silence. We kept `@Observes` and had the broadcaster spawn a virtual thread per peer for the actual HTTP call. Same effect without the CDI spec complication.

Also mid-branch: the local engine snapshot updated (engine#390), changing the `WorkerProvisioner` return type and how completed workers are named in ledger entries. We adapted `JpaCaseLineageQuery` to use a sequence-number-based lookup instead of timestamp, which is strictly better for concurrent workers. The engine compatibility work added a commit but didn't change the branch's purpose.

520 tests pass. The channel backend now has one registration path, and it works on peers.
