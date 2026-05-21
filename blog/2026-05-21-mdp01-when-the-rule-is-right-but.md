---
layout: post
title: "When the Rule Is Right but the Fix Is Wrong"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [claudony]
tags: [qhorus, cdi, architecture, reactive]
---

PLATFORM.md had a boundary rule: don't call `ReactiveQhorusMcpTools` from internal service code. `ReactiveQhorusMcpTools` is the MCP protocol dispatch layer for Claude Code — calling it from `MeshResource` couples a REST endpoint to an external protocol layer, complete with `@WrapBusinessError` exception semantics that internal consumers have no business unwrapping.

The rule was right about what not to do. It was wrong about what to do instead.

The prescribed fix was "inject `ReactiveChannelService` / `ReactiveMessageService` directly." Reasonable at first glance. But `ReactiveChannelService.listAll()` returns raw `Channel` entities — no message counts. Building `ChannelView` (with a count per channel) means injecting `ReactiveMessageStore` too, which is the store layer, not the service layer. We'd have fixed the protocol coupling by introducing a worse one: a REST resource reaching through two abstraction tiers into the storage implementation.

I pushed back on the rule mid-design. The right answer wasn't any of the existing integration points — it was a missing one.

So we added `QhorusDashboardService` to Qhorus: a proper consumer integration tier sitting between raw entity services and the MCP dispatch layer. It owns `listChannels()` (parallel message-count fan-out via `Uni.join()`), `listInstances()`, `getTimeline()`, `getFeed()`, and `sendHumanMessage()`. `MeshResource` now injects exactly one thing. No `@Blocking`, no timeouts, no `ToolCallException` unwrapping — just `Uni<T>` all the way up.

The implementation surfaced two CDI problems neither of us had seen coming.

The first: we annotated `QhorusDashboardService` with `@Alternative @IfBuildProperty`. The build property gates the bean's inclusion in the CDI context at augmentation time. `@Alternative` marks a bean as deselected by default — active only with `@Priority` or an explicit `quarkus.arc.selected-alternatives` entry. Together, they behave in a way nothing in the docs describes: the build property activates all the bean's dependencies but not the bean itself. A consumer app that sets only the gating property gets a deployment failure with all dependencies present and the service absent.

A separate Claude instance — reviewing the final diff — caught it. Without an interface to override, `@Alternative` is redundant: `@IfBuildProperty` alone handles activation, which is exactly how all other reactive services in Qhorus are annotated. The fix was removing `@Alternative` entirely.

The second problem was subtler. `ReactiveMessageService.send()` uses `Panache.withTransaction()` — the zero-argument form, which always targets the default persistence unit. Claudony configures only a named `qhorus` datasource, no default. The Qhorus library's own tests define a default H2, so they passed; the failure only appeared in Claudony integration: messages posted via `MeshResource.postMessage()` weren't showing up in the subsequent timeline query. The fix is `Panache.withTransaction("qhorus", ...)` — a named-PU overload that isn't prominently documented anywhere.

The boundary rule now names three integration points instead of one: `QhorusDashboardService` for dashboard/UI consumers needing composed views, entity services for SPI implementations and background workers, `ChannelBackend`/`MessageObserver` SPIs for reactive event handling. The distinction was always there; the rule just hadn't articulated it yet.
