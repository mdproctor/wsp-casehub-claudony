# Design Journal — epic-gateway-reliability

### 2026-05-19 · Scope and architecture direction · §Architecture

Epic originally scoped to #100 (catch-up on reconnect), #101 (re-register on restart),
#102 (fleet registration). Brainstorming discovered all three are retro-fits on top of
`ClaudonyChannelBackend` (#98), which does not yet exist. Added #98 to epic scope.

Architectural direction for #98: `ClaudonyChannelBackend.post()` fires a CDI event
(`Event<ChannelMessageEvent>`). A new SSE endpoint `/api/mesh/channels/{name}/events`
streams those events to the browser via Mutiny `Multi`. Panel JS replaces the 3-second
`pollChannel()` timer with an `EventSource` connection. Latency drops from ≤3s to near-zero.

Catch-up (#100) folds into the SSE endpoint: on connect, client sends `?after=lastId`;
server bursts missed messages before going live. `chLastId` already tracked in `terminal.js`
and must be preserved when push replaces polling.

Brainstorming not yet complete — design not approved. Next session resumes here.
