# Claudony Workbench — Rich Conversation Integration

**Issue:** #175
**Date:** 2026-07-19
**Status:** Approved

## Problem

Claudony's terminal page uses a three-column layout (worker-panel + terminal + channel-panel) that is an awkward middle ground — not simple enough for fleet management, not rich enough for case work. The channel-panel composes only `<channel-feed>` and `<channel-input>` from blocks-ui, discarding the rich conversation model that Qhorus already provides (replies, commitments, correlation, artifacts). The chat-app proves these capabilities work; Claudony should absorb them.

## Two UI Configurations

Claudony serves two distinct user intents from the same codebase:

- **Fleet manager** — session grid, click-to-enter terminals, session lifecycle. Terminal-first, minimal chrome. The existing dashboard + terminal page. Stays as-is.
- **Workbench** — case-aware, conversation-rich, task-driven. Terminal is one panel in a workspace alongside conversation, tasks, correlation, artifacts. New for this issue.

The natural dividing line is case context. Standalone sessions (no `caseId`) get fleet mode. Case-bound sessions get the workbench. The user can toggle between them.

## Architecture

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
| `<claudony-task-panel>` | chat-app `qhorus-task-panel` | Commitment tracking — active/overdue/completed | Yes — pure Qhorus types |
| `<claudony-correlation-panel>` | chat-app `qhorus-correlation-panel` | Conversation chain visualization | Yes — pure Qhorus types |
| `<claudony-artifact-panel>` | chat-app `qhorus-artifact-panel` | Artifact browser with history | Yes — pure Qhorus types |

Absorbed panels are built with zero Claudony-specific imports — `QhorusMessage`, `CommitmentState`, `ArtefactRef` types as inputs, `pages-event` for output. Proven in Claudony, then promoted to blocks-ui.

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

### Terminal Integration

The current `terminal.ts` entry point sets up xterm.js imperatively. This is extracted into a `terminal-controller.ts` module:

```typescript
export function attachTerminal(container: HTMLElement, sessionId: string): TerminalHandle
export interface TerminalHandle {
  dispose(): void
  resize(cols: number, rows: number): void
}
```

- The workbench calls `attachTerminal()` in `firstUpdated()` on a `<div>` ref
- The existing fleet-mode terminal page calls the same module
- No duplication of WebSocket, history replay, or resize logic

### Entry Point Routing

The terminal page (`session.html` / `terminal.ts`) detects case context and conditionally renders:
- **No `caseId`** → existing `<terminal-workspace>` (fleet mode, unchanged)
- **Has `caseId`** → `<claudony-workbench>` (workbench mode)

Single page, conditional render. The workbench is a LitElement — it composes cleanly.

## Backend Enrichment

### Widen `PostMessageRequest`

Current: `record PostMessageRequest(String content, String type)`

New fields: `inReplyTo` (Long), `correlationId` (String), `artefactRefs` (List), `target` (String), `deadline` (Instant), `topic` (String).

All passed through `QhorusDashboardService.sendHumanMessage()` → `MessageDispatch.builder()`. No new Qhorus SPIs — just surfacing what `ReactiveMessageService.dispatch()` already handles.

### Widen `sendHumanMessage()`

Current signature takes `(channelName, sender, type, content)`. New signature takes additional fields from the widened request, passes them to `MessageDispatch.builder()`.

### Add commitment endpoint

`GET /api/mesh/channels/{name}/commitments` — surfaces commitments for a channel via `ReactiveCommitmentStore`. The task panel needs this data.

### Enrich timeline entries

Verify `QhorusEntityMapper.toTimelineEntry()` carries through: `inReplyTo`, `correlationId`, `artefactRefs`, `target`, `replyCount`, `commitmentId`, `deadline`, `topic`. Fix any fields that are currently stripped.

## Frontend Adapter Enrichment

### Widen `TimelineEntry`

Add fields: `in_reply_to`, `correlation_id`, `artefact_refs`, `target`, `reply_count`, `commitment_id`, `deadline`, `topic`.

### `toQhorusMessage()` maps all fields

Stop defaulting `correlationId`, `inReplyTo`, `artefactRefs`, `target`, `replyCount` to empty. The `QhorusMessage` type from blocks-ui already has all these fields.

### Add `CommitmentRecord` type

Same shape as the chat-app's type. The workbench fetches commitments from the new endpoint and manages them as reactive state, passing to task/correlation panels as properties.

## Scope Boundaries

**In scope:**
- `<claudony-workbench>` LitElement with data flow and event routing
- Terminal extraction into `terminal-controller.ts`
- Backend enrichment (PostMessageRequest, sendHumanMessage, commitment endpoint, timeline enrichment)
- Adapter enrichment (TimelineEntry, toQhorusMessage)
- Absorbed panels (task, correlation, artifact) built for blocks-ui promotion
- Conditional workbench/fleet rendering based on case context
- Tests: backend (MeshResource), adapter (vitest), E2E (workbench rendering)

**Out of scope (future issues):**
- Case browser (#176)
- Task inbox across channels (#176)
- General-purpose chat rooms (not case-scoped)
- Reactions and member/presence panels (later phase of #175)
- Responsive layouts for tablet/phone
- Promoting panels to blocks-ui (after proven in Claudony)

## Design Principles

1. **Composability over layout** — get the component contracts and data flow right; layout is CSS
2. **blocks-ui promotion path** — absorbed panels use only Qhorus types, no Claudony imports
3. **No new transport** — SSE + poll, not WebSocket. Works fine for Claudony's use case
4. **No new Qhorus SPIs** — just surface what `ReactiveMessageService.dispatch()` already handles
5. **Fleet mode untouched** — standalone sessions keep the current terminal-only experience
