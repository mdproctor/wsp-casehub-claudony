# Design Journal — issue-131-channeleventbus-push

### 2026-05-27 · §Architecture

The `channelEvents()` SSE endpoint previously polled `QhorusDashboardService.getTimeline()`
every 500 ms regardless of message activity. The model is replaced with a merged-signals
pipeline: `ChannelEventBus` push events (fired by `ClaudonyChannelBackend.post()` on every
gateway-delivered message) and a preference-driven heartbeat. A single
`transformToUniAndConcatenate` serialises all `getTimeline()` fetches, preventing duplicate
frames from concurrent calls with the same `lastSentId`. The heartbeat emits `"[]"` — an
empty JSON array that the frontend renders as a no-op — rather than an SSE comment, because
`Multi<String>` (the existing return type) cannot produce comment-only frames without introducing
the Jakarta SSE API. The `"[]"` heartbeat also serves as a bounded recovery mechanism for the
window between gateway backend registration and ChannelEventBus subscription establishment.
`emitOn(Infrastructure.getDefaultWorkerPool())` is applied to the push stream to shift
ChannelEventBus emissions off the Qhorus I/O thread, avoiding silent frame drops caused by
cross-context Vert.x Netty writes.

### 2026-05-27 · §Component Structure

`ChannelHeartbeatInterval` is added as a typed `SingleValuePreference` record, mirroring
`ChannelCursorStaleness`. It carries a `PreferenceKey` with namespace `claudony`, name
`channelHeartbeatIntervalSeconds`, and a default of 30 seconds. The interval is resolved once
per SSE connection from `PreferenceProvider` and baked into the heartbeat `Multi` for that
connection's lifetime. `MeshConfig` gains a `channelHeartbeatSeconds` field so the frontend
can observe the configured interval. `MeshResource` is annotated `@ApplicationScoped` for
CDI interceptor support (auth retrofit readiness, PP-20260526-d0b921).
