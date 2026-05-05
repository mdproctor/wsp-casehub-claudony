---
layout: post
title: "Server-sent events and two silent failures"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
---

The 3-second worker list poll is gone. The case worker panel now runs on
server-sent events — updates arrive when the server fires them.

## The circular dependency

The implementation started with a module constraint. The component that
observes worker state, `ClaudonyWorkerStatusListener`, lives in
`claudony-casehub`. The broadcaster that fans events to SSE clients lives in
`claudony-app`. Maven dependency order runs core → casehub → app — so casehub
cannot import from app.

The usual options are to restructure the modules or introduce an interface. CDI
events offer a third path. We defined `WorkerCaseLifecycleEvent` in
`claudony-core`, which both sides share. The listener fires it. The broadcaster
observes it. Quarkus wires the observer at build time; neither module knows about
the other's implementation.

## The strategy

The broadcaster delegates to a pluggable `CaseWorkerUpdateStrategy`: events-only
(push on CDI events only), hybrid (events plus a periodic heartbeat), or
registry-hooks (push on any `SessionRegistry` mutation, catching external kills
that bypass the lifecycle path). The default is hybrid with a 30-second heartbeat
— real-time for CaseHub-managed state changes, eventual correction for everything
else.

## Two silent failures

The panel showed empty. The server was emitting; the browser's
`EventSource.onmessage` was firing. `JSON.parse(event.data)` was failing silently.

Quarkus wraps `Multi<String>` SSE output automatically. Each item the Multi emits
becomes `data: <item>\n\n` on the wire. If the snapshot function returns
`"data: " + json + "\n\n"`, the browser receives `e.data = "data: [...]"` — prefix
still attached, `JSON.parse` failing. The fix is to return plain JSON from the
snapshot function and let Quarkus add the framing. This is the opposite of what
`MeshResource.events()` does, because that endpoint is consumed by a raw `fetch()`
reading the body as a string, not by `EventSource`. Two consumption patterns,
two different requirements.

The second failure had the same symptom and a different cause. The snapshot
serialised `SessionResponse`, which has `Instant` fields. The `ObjectMapper` was
a static `new ObjectMapper()` — no `JavaTimeModule`. Serialisation threw
`InvalidDefinitionException`, the catch block returned `[]`, and the panel stayed
empty. The fix is `@Inject ObjectMapper`, which gets Quarkus's configured instance
with all the Jackson modules registered.

Both would have been invisible without E2E tests checking the rendered panel. The
endpoint returned HTTP 200 with valid JSON the whole time.
