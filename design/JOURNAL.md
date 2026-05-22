# Design Journal — epic-gateway-reliability

### 2026-05-19 · Scope and architecture direction · §Architecture

Epic originally scoped to #100 (catch-up on reconnect), #101 (re-register on restart),
#102 (fleet registration). Brainstorming discovered all three are retro-fits on top of
`ClaudonyChannelBackend` (#98), which does not yet exist. Added #98 to epic scope.

Architectural direction for #98: `ClaudonyChannelBackend.post()` fires a CDI event
(`Event<ChannelMessageEvent>`). A new SSE endpoint `/api/mesh/channels/{name}/events`
streams those events to the browser via Mutiny `Multi`. Panel JS replaces the 3-second
`pollChannel()` timer with an `EventSource` connection. Latency drops from ≤3s to near-zero.

### 2026-05-21 · #100 implemented as polling catch-up (SSE deferred to #98) · §Architecture

The SSE-based catch-up brainstormed in May 2019 was deferred: it requires `ClaudonyChannelBackend`
(#98) which is not yet built. #100 is implemented as a polling-era fix within the existing
`pollChannel()` / `timeline?after=` API.

The core change: `chLastId` (single variable, reset to 0 on every `selectChannel()` call) is
replaced by a per-channel cursor map (`chCursors`) backed by `sessionStorage`. On panel reopen:
- **Fresh cursor** (<30 min): silently appends new messages; feed is not cleared, so the user
  sees a seamless continuation rather than a blank feed with only new messages.
- **Stale cursor** (≥30 min): shows a prompt — "Catch up" (incremental) or "Reload full history".
- **No cursor**: full history load as before.

The staleness threshold is served from `GET /api/mesh/config` via the platform preferences API
(see §Technology Stack). When `ClaudonyChannelBackend` and SSE land (#98), the cursor map
remains relevant — the `sessionStorage` cursor can become the `Last-Event-ID` sent on reconnect.

### 2026-05-22 · #98 + #101: ClaudonyChannelBackend implemented; SSE delivery and restart re-registration · §Architecture

`ClaudonyChannelBackend` (singleton `@ApplicationScoped`, implements `HumanObserverChannelBackend`)
is registered as a Qhorus gateway backend so `ChannelGateway.fanOut()` calls `post()` on every
channel message. Two key design decisions differ from the original brainstorm:

**Registration timing**: originally planned in `ClaudonyCaseChannelProvider.openChannel()`, but
`claudony-app` cannot inject `claudony-casehub` beans without creating a circular module dependency.
Registration happens instead at SSE subscribe time (`MeshResource.channelEvents()`) and at
server restart (`ServerStartup.bootstrapChannelBackends()`). Functionally equivalent — there is
nothing to push to until a browser has an open EventSource.

**Delivery mechanism**: `ClaudonyChannelBackend.post()` ticks `ChannelEventBus` (future
infrastructure for event-driven delivery), but the SSE endpoint uses a 500ms server-side tick
via `Multi.createFrom().ticks()` rather than `ChannelEventBus.subscribe()`. The emitter-based
approach had a cross-thread dispatch issue (Vert.x event loop thread vs. SSE response thread)
causing frames to be computed but not delivered. The 500ms tick polls `QhorusDashboardService
.getTimeline()` per active connection; worst-case latency is 500ms, within the < 2s E2E
threshold. True push delivery via `ChannelEventBus` is tracked in #131.

`ServerStartup.bootstrapChannelBackends()` runs after `bootstrapRegistry()` at startup: queries
all channels matching active session caseId prefixes and re-registers the backend for each,
restoring `ChannelGateway` fan-out after a JVM restart. Registration is idempotent
(`deregisterBackend()` before each `registerBackend()`). The panel switches from `pollChannel()`
(3s timer) to `EventSource('/api/mesh/channels/{name}/events?after=<cursor>')`, falling back to
polling on SSE error.

### 2026-05-22 · ChannelEventBus and ClaudonyChannelBackend as new server components · §Component Structure

Two new `@ApplicationScoped` beans in `claudony-app`:
- `ChannelEventBus` — in-process fan-out keyed by channel name; `subscribe(name)` returns
  `Multi<Integer>` (tick signals); `emit(name)` fires to all active subscribers; currently unused
  by the SSE endpoint but retained for future event-driven delivery (#131)
- `ClaudonyChannelBackend` — implements Qhorus `HumanObserverChannelBackend` SPI; `BACKEND_ID =
  "claudony-observer"` (stable constant); `post()` ticks `ChannelEventBus`; `open()`/`close()`
  are no-ops; one instance serves all channels

### 2026-05-22 · Real-time channel message delivery data flow · §Data Flows

Agent posts message → `QhorusMcpTools.sendMessage()` persists → `ChannelGateway.fanOut()` calls
`ClaudonyChannelBackend.post()` in a virtual thread → `ChannelEventBus.emit(name)` (tick, no SSE
subscriber yet) → SSE endpoint's 500ms tick fires → `QhorusDashboardService.getTimeline(name,
lastSentId, 50)` fetches new entries → serialized as bare JSON → RESTEasy Reactive wraps in
`data: [...]\n\n` SSE frame → browser `EventSource.onmessage` → `appendMessages(entries)` →
`chCursors` updated in `sessionStorage`. On SSE error the panel falls back to `pollChannel()`.

On server restart: `bootstrapChannelBackends()` re-registers `ClaudonyChannelBackend` for all
channels belonging to sessions with active caseIds, restoring the fan-out path before any
browser reconnects.

### 2026-05-21 · Platform preferences API introduced · §Technology Stack

Claudony now depends on `casehub-platform-api` and `casehub-platform-config`. The
`ChannelCursorStaleness` typed preference key (`claudony.channelCursorStalenessMinutes`,
default 30) is the first Claudony preference using the platform's typed preference API.
`GET /api/mesh/config` resolves it via `PreferenceProvider` and serves it to the frontend.
Future Claudony configuration knobs should follow the same pattern: a typed
`SingleValuePreference` record with a `PreferenceKey` constant.
