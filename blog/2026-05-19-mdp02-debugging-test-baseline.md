---
layout: post
title: "Debugging a Test Baseline"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: log
projects: [Claudony]
---

A full test run surfaced three things that needed fixing. One was trivial, one
took four attempts to understand, and one was a missed update from the previous
session's reactive migration.

The trivial one: `McpServerIntegrationTest` asserts exactly 60 MCP tools (8
Claudony + 52 Qhorus). A fresh SNAPSHOT of Qhorus had shed two tools since the
assertion was written. Updated to 58.

The interesting one was `MeshResourceInterjectionTest`. Nine errors, all `ToolCallException:
IllegalStateException: No current Vertx context found`. The `@BeforeEach` setup
was calling `tools.createChannel()` from the test thread — fine before, broken
now because `ReactiveChannelService.create()` gained a `Panache.withTransaction()`
wrapper in a newer Qhorus version.

We tried `vertx.executeBlocking()`. The error changed: `Can't get the context
safety flag: the current context is not a duplicated context`. Panache requires
a *duplicated* context, not a root one. `executeBlocking()` gives you the latter.

Rather than chase the context problem further, we read the `ReactiveChannelService`
source and found `create()` calls `channelStore.put()` inside the transaction —
the in-memory store handles it, no actual database involvement. We bypassed the
service entirely and wrote directly to `InMemoryChannelStore.put()`. Nine errors
became six failures.

Still failing, but now 404s. We dug into how `sendMessage()` resolves channels
and found it uses `blockingChannelService.findByName()` — the synchronous
`ChannelStore`, not the reactive one. Our channel was in the right store. But
the messages weren't.

`getChannelTimeline()` doesn't use `MessageStore` at all. It queries `Message.find()`
directly — a Panache static hitting H2. Messages written through `blockingMessageService`
land in `InMemoryMessageStore` (in-memory). Timeline reads from H2. Neither path
sees the other's data.

This is a Qhorus inconsistency: `sendMessage()` uses an abstracted store interface
for writes, `getChannelTimeline()` bypasses the abstraction entirely for reads.
In production both paths hit the same database and it doesn't matter. In tests
with in-memory alternatives it falls apart. We filed qhorus#173 — it was fixed
and verified within the session.

The last failure was `SessionLineageResourceTest` returning 500. After the
reactive SPI migration, `CaseLineageQuery.findCompletedWorkers()` now returns
`Uni<List<WorkerSummary>>`. The endpoint still called
`Response.ok(lineageQuery.findCompletedWorkers(caseUuid)).build()` — passing
the `Uni` object itself as the response entity. RESTEasy can't serialize a `Uni`,
so 500. Fix: `@Blocking` on the endpoint, `.await().indefinitely()` on the call.

475 tests passing, 0 failures.
