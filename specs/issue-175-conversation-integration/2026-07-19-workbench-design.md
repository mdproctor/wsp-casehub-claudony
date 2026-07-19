# Claudony Workbench — Rich Conversation Integration

**Issue:** #175
**Date:** 2026-07-19
**Status:** Approved
**Cross-ref:** chat-app `docs/specs/2026-07-18-dockable-contextual-panels-design.md` (panel architecture), chat-app `docs/specs/2026-07-19-topic-navigator-design.md` (topic support)

## Problem

Claudony's terminal page uses a three-column layout (worker-panel + terminal + channel-panel) that is an awkward middle ground — not simple enough for fleet management, not rich enough for case work. The channel-panel composes only `<channel-feed>` and `<channel-input>` from blocks-ui, discarding the rich conversation model that Qhorus already provides (replies, commitments, correlation, artifacts). The chat-app proves these capabilities work; Claudony should absorb them.

## Two UI Configurations

Claudony serves two distinct user intents from the same codebase:

- **Fleet manager** — session grid, click-to-enter terminals, session lifecycle. Terminal-first, minimal chrome. The existing dashboard + terminal page. Stays as-is.
- **Workbench** — case-aware, conversation-rich, task-driven. Terminal is one panel in a workspace alongside conversation, tasks, correlation, artifacts. New for this issue.

The natural dividing line is case context. Standalone sessions (no `caseId`) get fleet mode. Case-bound sessions get the workbench. The user can toggle between them.

## Architecture

### Relationship to chat-app Workbench

The chat-app's `qhorus-workbench.ts` (604 lines) is the reference implementation for the panel composition pattern. The claudony workbench adopts:

- **Dock strip model** — same `DockItem[]` array with pages `LayoutStore` persistence
- **Data flow pattern** — reactive `@state()` properties passed down, `pages-event` bubbling up
- **Panel rendering** — same `_renderPanel(panelId)` switch for task/correlation/artifact panels
- **Responsive layout** — desktop/tablet/phone modes with media queries, drawer infrastructure

The claudony workbench **differs** in:

- **Transport** — SSE + poll (not WebSocket), matching Claudony's existing infrastructure
- **Terminal integration** — xterm.js panel via `terminal-controller.ts` (chat-app has no terminal)
- **Case context** — case header, worker lineage, case-scoped channel auto-selection
- **Channel model** — Claudony connects to Qhorus channels via MeshResource REST/SSE; chat-app uses a custom SQLite backend with WebSocket dataset protocol

### Component: `<claudony-workbench>`

A LitElement composition root that replaces `<terminal-workspace>` + `<channel-panel>` + `<worker-panel>` for case-bound sessions. It manages data flow and event routing; layout is a CSS concern that can change independently.

**Children — from blocks-ui (existing dependencies):**

| Component | Replaces | Role |
|-----------|----------|------|
| `<channel-nav>` | Native `<select>` dropdown | Channel list with unread indicators |
| `<channel-feed>` | Already used | Message display with threading, reactions |
| `<channel-input>` | Already used | Message composition with speech act selector |

**Children — absorbed from chat-app (new, blocks-ui candidates):**

| Component | Source | Role | blocks-ui candidate? |
|-----------|--------|------|---------------------|
| `<claudony-task-panel>` | chat-app `qhorus-task-panel` | Commitment tracking — active/overdue/completed | Yes — uses `CommitmentRecord` + `QhorusMessage` |
| `<claudony-correlation-panel>` | chat-app `qhorus-correlation-panel` | Conversation chain visualization | Yes — uses `QhorusMessage` |
| `<claudony-artifact-panel>` | chat-app `qhorus-artifact-panel` | Artifact reference viewer (v1: metadata only, no content fetching) | Yes — uses `ArtefactRef` |

Absorbed panels use only types that are either already in blocks-ui (`QhorusMessage`, `CommitmentState`, `ArtefactRef`) or are blocks-ui promotion candidates (`CommitmentRecord`). No Claudony-specific imports. The chat-app's dockable panels spec defines `CommitmentRecord` for promotion to `@casehubio/blocks-ui-channel-activity/types.ts`; this spec depends on that promotion landing first. If it hasn't landed when implementation begins, define an identical local type in the Claudony adapter layer.

**Children — Claudony-specific:**

| Component | Role |
|-----------|------|
| Case header | Role, status, elapsed time (from existing channel-panel) |
| Worker list | Worker switching for multi-worker cases (from existing worker-panel) |
| Terminal | xterm.js via shared `terminal-controller.ts` module |

### Data Flow

```
MeshResource (SSE + poll)
    ↓ channels, messages, commitments
<claudony-workbench> (reactive state)
    ↓ properties
<channel-nav>  <channel-feed>  <task-panel>  <correlation-panel>  <artifact-panel>
    ↑ pages-event (send-message, select-channel, message-selected, etc.)
<claudony-workbench> (event routing → MeshResource POST)
```

- The workbench owns the connection to MeshResource — SSE with poll fallback (same transport as today)
- Channels, messages, and commitments are `@state()` properties, passed down to children
- Events bubble up from blocks-ui components via `pages-event`
- The workbench dispatches actions to MeshResource and updates state

#### SSE Strategy — Per-Channel, Close-and-Reopen

The workbench maintains a single SSE connection to `/api/mesh/channels/{name}/events` for the active channel. On channel switch:

1. Close the current `EventSource`
2. Open a new `EventSource` for the selected channel
3. Load timeline history via `GET /api/mesh/channels/{name}/timeline`

This matches the existing `channel-panel.ts` strategy — no connection churn beyond what already exists. The aggregate `/api/mesh/events` endpoint is not used; per-channel SSE gives cleaner message boundaries.

#### Commitment Freshness — Re-fetch on SSE Tick

When the SSE stream delivers new timeline entries, the workbench re-fetches commitments via `GET /api/mesh/channels/{name}/commitments`. This adds one HTTP round-trip per SSE tick that contains new messages — acceptable because:

- Commitments change infrequently relative to message delivery
- The SSE polling interval is 500ms, but most ticks return empty (no new messages)
- Re-fetch is skipped when the SSE tick contains no new messages

The commitment response is small (typically <20 records per channel) so the round-trip cost is negligible.

#### channel-panel.ts Logic Redistribution

The existing `channel-panel.ts` (484 lines) is replaced by the workbench. Its responsibilities move as follows:

| Responsibility | Current owner | New owner |
|---------------|---------------|-----------|
| SSE EventSource lifecycle | channel-panel | `<claudony-workbench>` composition root |
| Cursor persistence in sessionStorage | channel-panel | `<claudony-workbench>` composition root |
| Stale cursor detection + refresh prompt | channel-panel | `<claudony-workbench>` composition root |
| Mesh config fetch (staleness settings) | channel-panel | `<claudony-workbench>` composition root |
| Channel loading and listing | channel-panel | `<claudony-workbench>` composition root |
| Channel auto-selection (`case-{caseId}/work`) | channel-panel | `<claudony-workbench>` composition root |
| Message posting via fetch | channel-panel | `<claudony-workbench>` event routing |
| Case header rendering (role, status, elapsed) | channel-panel | Claudony-specific case header child |
| Worker lineage loading and display | channel-panel | Claudony-specific worker list child |
| Allowed types computation | channel-panel | `<claudony-workbench>` passes to `<channel-input>` |

### Terminal Integration

The current `terminal.ts` entry point sets up xterm.js imperatively via `PagesTerminal` from `@casehubio/pages-component-terminal`. This is extracted into a `terminal-controller.ts` module that encapsulates all terminal concerns:

```typescript
export function attachTerminal(
  container: HTMLElement, sessionId: string, opts?: { proxyPeer?: string }
): TerminalHandle

export interface TerminalHandle {
  dispose(): void
  resize(cols: number, rows: number): void
  sendInput(text: string): void
  switchSession(sessionId: string, opts?: { proxyPeer?: string }): void
}
```

- `dispose()` — tears down the `PagesTerminal` and WebSocket connection
- `resize(cols, rows)` — handles BOTH xterm.js resize AND the tmux resize REST call (`POST /api/sessions/{id}/resize`). The caller doesn't need to know about tmux.
- `sendInput(text)` — forwards keyboard input to the terminal (required for the key-bar on touch devices)
- `switchSession(sessionId)` — reconstructs the WebSocket URL and calls `PagesTerminal.configure({ wsUrl })` to reconnect to a different session. Used during worker switching.

The module internally constructs WebSocket URLs (`wss://{host}/ws/{sessionId}/{cols}/{rows}`) and handles proxy peer routing (`/ws/proxy/{peer}/{sessionId}/...`). These are terminal implementation details that the workbench doesn't need to see.

- The workbench calls `attachTerminal()` in `firstUpdated()` on a `<div>` ref
- The existing fleet-mode terminal page calls the same module
- No duplication of WebSocket, history replay, resize logic, or URL construction

#### terminal-workspace.ts Logic Redistribution

The existing `terminal-workspace.ts` (161 lines) is replaced by the workbench for case-bound sessions (fleet mode keeps it as-is). Its responsibilities move as follows:

| Responsibility | Current owner | New owner |
|---------------|---------------|-----------|
| WebSocket URL construction (incl. proxy peer) | terminal-workspace | `terminal-controller.ts` module (internal) |
| `PagesTerminal` creation + configuration | terminal-workspace | `terminal-controller.ts` via `attachTerminal()` |
| tmux resize REST call (`POST /api/sessions/{id}/resize`) | terminal-workspace `handleResize()` | `terminal-controller.ts` via `TerminalHandle.resize()` |
| Key-bar input forwarding (`sendInput`) | terminal-workspace `wireEvents()` | `<claudony-workbench>` catches `key-pressed` → `TerminalHandle.sendInput()` |
| Worker switch — terminal reconnection | terminal-workspace `handleWorkerSwitch()` | `<claudony-workbench>` catches `worker-selected` → `TerminalHandle.switchSession()` |
| Worker switch — URL update (`history.replaceState`) | terminal-workspace `handleWorkerSwitch()` | `<claudony-workbench>` event routing |
| Worker switch — panel re-configuration | terminal-workspace `handleWorkerSwitch()` | `<claudony-workbench>` updates `@state()` props (channels, messages re-fetched for new session) |
| Worker switch — `session-changed` event dispatch | terminal-workspace `handleWorkerSwitch()` | `<claudony-workbench>` dispatches `session-changed` for header |
| DOM composition (worker-panel + terminal + channel-panel) | terminal-workspace `render()` | `<claudony-workbench>` LitElement `render()` |

#### Worker Switch Lifecycle

When the worker list dispatches `worker-selected` with a new `sessionId`:

1. `TerminalHandle.switchSession(newSessionId)` — terminal reconnects to new session's WebSocket
2. Workbench updates `_sessionId` state → triggers re-fetch of channels, messages, commitments for the new session
3. `history.replaceState()` — URL updated without page reload
4. Workbench dispatches `session-changed` event for the page header to update session name
5. Channel-switch resets apply (`_selectedMessageId`, `_replyTo`, `_selectedArtefactRef` cleared)

### Entry Point Routing

The terminal page (`session.html` / `terminal.ts`) detects case context and conditionally renders:
- **No `caseId`** → existing `<terminal-workspace>` (fleet mode, unchanged)
- **Has `caseId`** → `<claudony-workbench>` (workbench mode)

The workbench is loaded via dynamic `import()` to avoid bundling its dependency tree (task/correlation/artifact panels, blocks-ui components) into fleet-mode sessions that never use it:

```typescript
if (session.caseId) {
  const { ClaudonyWorkbench } = await import('./components/claudony-workbench.js');
  // render workbench
} else {
  // render existing terminal-workspace — no workbench code loaded
}
```

This keeps the fleet-mode `terminal` chunk lean — only xterm.js and the existing terminal-workspace.

### Threading Interaction Model

The workbench supports reply-based threading for speech acts that require `inReplyTo`:

1. **Click a message** in `<channel-feed>` → dispatches `channel:message-selected` event
2. **Workbench catches event** → sets `_replyTo = { messageId, senderName }` and `_selectedMessageId`
3. **`<channel-input>`** receives `replyTo` prop → shows reply indicator above textarea
4. **User sends** → `PostMessageRequest` includes `inReplyTo` (the selected message ID) and `correlationId` (inherited from the selected message's correlation chain)
5. **Cancel reply** → user clicks dismiss on reply indicator → clears `_replyTo`

**Channel switch resets:** When the user switches channels, the workbench clears `_selectedMessageId`, `_replyTo`, and `_selectedArtefactRef`. These reference messages/artifacts in the previous channel and would cause stale rendering (correlation panel for a nonexistent message, reply indicator for a message in another channel).

This is the same interaction model as the chat-app workbench's `_onChatEvent` handler for `MESSAGE_SELECTED`.

## Backend Enrichment

### Widen `PostMessageRequest`

Current: `record PostMessageRequest(String content, String type)`

New fields: `inReplyTo` (Long), `correlationId` (String), `artefactRefs` (List), `target` (String), `deadline` (Instant), `topic` (String).

All passed through `QhorusDashboardService.sendHumanMessage()` → `MessageDispatch.builder()`.

#### Validation

`MeshResource.postMessage()` must validate speech-act-specific field requirements before calling `sendHumanMessage()`:

| Message type | Required fields | Validation |
|-------------|----------------|------------|
| RESPONSE | `inReplyTo`, `correlationId` | 400 if either is missing |
| DONE, DECLINE | `inReplyTo`, `correlationId` | 400 if either is missing |
| HANDOFF | `inReplyTo`, `correlationId`, `target` | 400 if any is missing |
| EVENT | — | 400 if `content` is non-null |
| STATUS, QUERY, COMMAND | — | No additional requirements |

FAILURE is intentionally excluded from `VALID_HUMAN_TYPES` (already enforced — humans signal decisions, not automated failure states).

**Error mapping fix:** The current `MeshResource.postMessage()` maps `IllegalArgumentException` from the builder to HTTP 404. This is incorrect — `IllegalArgumentException` from `MessageDispatch.builder().build()` is a validation error (400), while `IllegalArgumentException` from `sendHumanMessage()` for channel-not-found should be 404. The fix: perform field validation explicitly in `postMessage()` before calling `sendHumanMessage()`, returning 400 for invalid fields. The remaining `IllegalArgumentException` from `sendHumanMessage()` (channel not found) correctly maps to 404.

### Widen `sendHumanMessage()`

Current signature takes `(channelName, sender, type, content)`. New signature:

```java
public Uni<HumanMessageResult> sendHumanMessage(
        String channelName, String sender, MessageType type, String content,
        Long inReplyTo, String correlationId, List<ArtefactRef> artefactRefs,
        String target, Instant deadline, String topic)
```

All new fields are `@Nullable` and passed through to `MessageDispatch.builder()`.

### Add commitment endpoint

`GET /api/mesh/channels/{name}/commitments` — returns all commitments for a channel, regardless of state (OPEN, FULFILLED, FAILED, DECLINED, EXPIRED, etc.).

**New SPI method required:** `ReactiveCommitmentStore` needs a `findByChannel(UUID channelId)` method returning `Uni<List<Commitment>>`. The existing interface has per-state methods (`findByState`, `findOpenByObligor`) but no method that returns all commitments for a channel. Calling `findByState` for each possible state would work but is wasteful — a single `findByChannel` query is the right design.

The response maps `Commitment` records to a JSON representation including: `id`, `correlationId`, `state`, `requester`, `obligor`, `expiresAt`, `acknowledgedAt`, `resolvedAt`, `delegatedTo`, `createdAt`.

### Enrich timeline entries

`QhorusEntityMapper.toTimelineEntry()` currently includes for MESSAGE entries: `id`, `type`, `created_at`, `sender`, `message_type`, `content`, `correlation_id`. Add the missing fields:

| Field | Source on `Message` | Currently included? |
|-------|-------------------|-------------------|
| `in_reply_to` | `m.inReplyTo()` | No — add |
| `artefact_refs` | `m.artefactRefs()` | No — add |
| `target` | `m.target()` | No — add |
| `reply_count` | `m.replyCount()` | No — add |
| `deadline` | `m.deadline()` | No — add |
| `topic` | `m.topic()` | No — add |

Note: `commitmentId` is NOT a field on the `Message` record. Commitment lookup is a client-side join — the frontend resolves commitment state from its local commitment data using `correlationId` as the join key (see §Frontend Adapter Enrichment).

## Frontend Adapter Enrichment

### Widen `TimelineEntry`

Add fields: `in_reply_to`, `correlation_id` (already present), `artefact_refs`, `target`, `reply_count`, `deadline`, `topic`.

Note: `commitment_id` is intentionally excluded from `TimelineEntry`. Commitment data flows separately via the commitment endpoint; the frontend joins commitments to messages using `correlationId`.

### `toQhorusMessage()` maps all fields

The current adapter defaults `correlationId`, `inReplyTo`, `artefactRefs`, `target`, `replyCount`, `topic` to empty/zero. Map them from `TimelineEntry`:

```typescript
return {
  // ... existing fields ...
  correlationId: entry.correlation_id,
  inReplyTo: entry.in_reply_to ? String(entry.in_reply_to) : undefined,
  artefactRefs: entry.artefact_refs ?? [],
  target: entry.target,
  replyCount: entry.reply_count ?? 0,
  topic: entry.topic ?? '',
  deadline: entry.deadline,
};
```

### Add `CommitmentRecord` type

Shape (matching the chat-app's type, defined in `@casehubio/blocks-ui-channel-activity/types.ts` per the dockable panels spec):

```typescript
interface CommitmentRecord {
  state: CommitmentState;
  deadline?: string;
  acknowledgedAt?: string;
  createdAt: string;
  updatedAt: string;
}
```

If the blocks-ui promotion hasn't landed when implementation begins, define an identical local type in `src/main/webui/src/util/channel-adapter.ts`.

#### Field mapping from Qhorus `Commitment` record

The Qhorus `Commitment` Java record does not have `updatedAt` or `deadline` fields directly. The commitment endpoint maps as follows:

| `CommitmentRecord` field | Source on Qhorus `Commitment` | Notes |
|-------------------------|------------------------------|-------|
| `state` | `commitment.state()` | Direct mapping |
| `deadline` | `commitment.expiresAt()` | Name difference — Qhorus uses `expiresAt`, chat-app uses `deadline` |
| `acknowledgedAt` | `commitment.acknowledgedAt()` | Direct mapping |
| `createdAt` | `commitment.createdAt()` | Direct mapping |
| `updatedAt` | Synthetic: `max(resolvedAt, acknowledgedAt, createdAt)` | Qhorus has no `updatedAt` — derived from the most recent state-change timestamp |

The synthetic `updatedAt` is computed in the MeshResource endpoint handler, not in the Qhorus layer. This keeps the adapter mapping self-contained and avoids modifying the Qhorus `Commitment` record.

### Artifact panel — v1 stub resolver

The chat-app's `qhorus-artifact-panel` takes a `resolveArtifact?: (ref: ArtefactRef) => Promise<{content: string, language?: string}>` callback for content fetching. In v1, the Claudony workbench provides a stub resolver that returns `{content: ref.label, language: undefined}` — the panel renders artifact metadata (type badge, label, URI with copy button) without fetching real content. This matches the chat-app's v1 approach per the dockable panels spec §6.

Content resolution by artifact type requires different strategies (`channel:` URIs → internal navigation, `http:` URIs → external fetch, `case:` URIs → case API) and is deferred to a follow-up issue. The artifact panel is still useful in v1: it surfaces what artifacts are referenced in a conversation, even without inline content display.

### Client-side commitment join

The workbench fetches commitments from `GET /api/mesh/channels/{name}/commitments` and stores them as `Map<string, CommitmentRecord>` keyed by `correlationId`. Panels that need commitment context (task panel, correlation panel) receive this map as a property. They join messages to commitments using the message's `correlationId` — no `commitmentId` field needed on the timeline entry.

## Scope Boundaries

**In scope:**
- `<claudony-workbench>` LitElement with data flow and event routing
- Terminal extraction into `terminal-controller.ts`
- Backend enrichment (PostMessageRequest, sendHumanMessage, commitment endpoint, timeline enrichment)
- `ReactiveCommitmentStore.findByChannel()` SPI addition
- `MeshResource.postMessage()` validation and error mapping fix
- Adapter enrichment (TimelineEntry, toQhorusMessage, CommitmentRecord)
- Absorbed panels (task, correlation, artifact) built for blocks-ui promotion
- Conditional workbench/fleet rendering based on case context
- Threading interaction model (click-to-reply, auto-populate inReplyTo/correlationId)

**Tests:**

| Area | Framework | Key scenarios |
|------|-----------|---------------|
| `MeshResource` | JUnit (Quarkus) | Widened `PostMessageRequest` validation; speech-act field requirements (RESPONSE without inReplyTo → 400); commitment endpoint returns all states; error mapping (builder validation → 400, channel not found → 404) |
| `sendHumanMessage()` | JUnit | New fields passed through to `MessageDispatch.builder()`; builder validation for RESPONSE/DONE/HANDOFF types |
| `toTimelineEntry()` | JUnit | New fields included in output; commitmentId NOT in output |
| Adapter (`toQhorusMessage`) | vitest | All new fields mapped from TimelineEntry; defaults for missing fields |
| `CommitmentRecord` mapping | vitest | Commitment endpoint response → `Map<string, CommitmentRecord>` |
| Workbench rendering | Playwright E2E | Workbench renders for case-bound sessions; fleet mode for standalone; channel switch closes/opens SSE; commitment panel shows data |
| xterm.js | Playwright E2E | Terminal renders in workbench layout; focus, resize work alongside panels |

Existing test baseline (587 tests) is expected to grow, not shrink.

**Out of scope (future issues):**
- Case browser (#176)
- Task inbox across channels (#176)
- General-purpose chat rooms (not case-scoped) — track as new issue
- Reactions and member/presence panels (later phase of #175) — track as new issue
- Responsive layouts for tablet/phone — track as new issue
- Promoting panels to blocks-ui (after proven in Claudony) — track as new issue

Deferred items without issue numbers must be filed as GitHub issues before this spec's implementation begins.

## Prerequisites

| Prerequisite | Repo | Dependency |
|-------------|------|------------|
| `ReactiveCommitmentStore.findByChannel(UUID)` | casehub-qhorus | Must be added to the SPI interface (`casehub-qhorus-api`), implemented in `InMemoryReactiveCommitmentStore` (`casehub-qhorus-persistence-memory`), and implemented in `ReactiveJpaCommitmentStore` (`casehub-qhorus-runtime`). Without this, MeshResource's commitment endpoint has no method to call. |
| `CommitmentRecord` in blocks-ui (optional) | casehub-blocks-ui | Per chat-app dockable panels spec. If not landed, use local identical type (fallback documented in §Frontend Adapter Enrichment). |

Implementation of the commitment endpoint is blocked on the Qhorus SPI addition. The rest of this spec (workbench component, terminal extraction, timeline enrichment, adapter enrichment) can proceed in parallel.

## Design Principles

1. **Composability over layout** — get the component contracts and data flow right; layout is CSS
2. **blocks-ui promotion path** — absorbed panels use only types that are already in blocks-ui or are blocks-ui promotion candidates (`CommitmentRecord`). No Claudony-specific imports in panel code.
3. **No new transport** — SSE + poll, not WebSocket. Works fine for Claudony's use case
4. **Minimal new Qhorus SPIs** — message dispatch uses `ReactiveMessageService.dispatch()` as-is. The one new SPI method is `ReactiveCommitmentStore.findByChannel()`, required to surface commitment data that the existing interface doesn't provide in a single query.
5. **Fleet mode untouched** — standalone sessions keep the current terminal-only experience
