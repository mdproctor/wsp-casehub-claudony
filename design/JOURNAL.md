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

### 2026-05-21 · Platform preferences API introduced · §Technology Stack

Claudony now depends on `casehub-platform-api` and `casehub-platform-config`. The
`ChannelCursorStaleness` typed preference key (`claudony.channelCursorStalenessMinutes`,
default 30) is the first Claudony preference using the platform's typed preference API.
`GET /api/mesh/config` resolves it via `PreferenceProvider` and serves it to the frontend.
Future Claudony configuration knobs should follow the same pattern: a typed
`SingleValuePreference` record with a `PreferenceKey` constant.
