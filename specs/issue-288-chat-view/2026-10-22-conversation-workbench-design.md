# Conversation Workbench — Shared Composition for Channel UI

**Issue:** casehubio/claudony#288
**Date:** 2026-10-22
**Status:** Draft

## Problem

Claudony and chat-app both compose the same mosaic channel-activity controllers
and components (feed, nav, input, member panel, task panel, correlation panel,
artifact panel) but wire them independently. Chat-app's `qhorus-workbench.ts`
(~800 lines) and claudony's `claudony-workbench.ts` (~600 lines) duplicate
controller instantiation, event routing, push connection setup, dock management,
and layout logic. Every mosaic feature addition requires parallel wiring in both
apps.

## Solution

Add a `<conversation-workbench>` LitElement to casehub-mosaic's channel-activity
package. It composes all existing mosaic controllers and components into a
responsive, full-viewport conversation UI. App-specific concerns (auth, message
delivery, channel filtering, space management) are injected via a configuration
object with callback functions.

Apps that need a different layout (like claudony's terminal-primary workbench)
continue to compose individual mosaic components directly. The workbench is the
default composition — not the only one.

## Configuration Interface

```typescript
interface ConversationWorkbenchConfig {
  // Auth — wraps every internal REST call (channel list, members, reactions)
  fetch: (url: string, init?: RequestInit) => Promise<Response>;

  // Real-time push connection
  pushUrl: string;

  // Identity — used as currentActorId on channel-feed
  currentActorId: string;

  // REST base for mosaic controllers (channel list, reactions, members)
  restBase: string;  // e.g. '/api' or '/api/mesh'

  // Channel scoping (optional — when absent, all channels shown)
  channelFilter?: (channel: QhorusChannel) => boolean;

  // Message operations
  sendMessage: (payload: SendMessagePayload) => Promise<void>;
  markRead?: (channelId: string, lastReadMessageId: string) => Promise<void>;

  // Space management (optional — UI hidden when all four are absent)
  createSpace?: (name: string) => Promise<void>;
  renameSpace?: (spaceId: string, newName: string) => Promise<void>;
  deleteSpace?: (spaceId: string) => Promise<void>;
  moveChannelToSpace?: (channelId: string, spaceId: string, position?: number) => Promise<void>;

  // Message corrections (optional — correction/retraction UI hidden when absent)
  correctMessage?: (channelId: string, messageId: string, content: string) => Promise<void>;
  retractMessage?: (channelId: string, messageId: string, reason?: string) => Promise<void>;

  // Layout persistence (optional — defaults used when absent)
  layoutStore?: {
    load(key: string): Promise<LayoutState | null>;
    save(key: string, state: LayoutState): Promise<void>;
  };
}
```

## What the Workbench Owns

| Concern | Implementation |
|---------|----------------|
| Controller instantiation | Creates PushController, ChannelStateController, MessagingController, MembershipController, ReactionController, CommitmentController |
| Push connection | Connects to `config.pushUrl` via EventConnection, routes ops to PushController |
| Event routing | Listens for `pages-event`, dispatches to controllers, delegates app operations to config callbacks |
| Responsive layout | Three breakpoints: desktop (≥1280px), tablet (768–1279px), phone (<768px) |
| Desktop layout | Dock strip + toggleable left panels (nav, tasks) + main chat + right panels (members, correlation, artifacts, settings) |
| Tablet layout | Tab-switcher sidebar + main chat |
| Phone layout | Slide-out drawers + header bar + main chat + swipe gestures |
| Dock panel management | Toggle, persist state via layoutStore |
| Drag-resize panels | Nav and member panel widths adjustable, persisted |
| Connection banner | Reconnecting/disconnected state display |
| Topic bar | Shown when channel has multiple topics |
| Commitment visualization | Range decorations, highlight bars on messages |
| Settings panel | Compact density toggle, theme picker |

## What Apps Provide

| Concern | Mechanism |
|---------|-----------|
| Authentication | `config.fetch` wrapping function |
| Message posting | `config.sendMessage` callback |
| Read cursor tracking | `config.markRead` callback (optional) |
| Space CRUD | Four optional callbacks; UI hidden when absent |
| Message corrections | Two optional callbacks; UI hidden when absent |
| Channel scoping | `config.channelFilter` predicate (optional) |
| Layout persistence | `config.layoutStore` (optional) |
| Push URL | `config.pushUrl` string |
| Actor identity | `config.currentActorId` string |

## Component API

```html
<conversation-workbench .config=${workbenchConfig}></conversation-workbench>
```

Single `config` property containing all configuration. No individual attributes
for each setting — the config object is the interface.

## Claudony Integration

### New component: `claudony-chat-panel.ts`

Thin shell (~50 lines) that creates a `ConversationWorkbenchConfig` with
claudony-specific values:

- `fetch`: claudony's `fetchWithAuth` (cookie-based session auth)
- `pushUrl`: `${wsProto}//${location.host}/ws/push`
- `currentActorId`: `'human'`
- `restBase`: `'/api'`
- `sendMessage`: POST to `/api/mesh/channels/{name}/messages`
- `channelFilter`: optional, based on `namespace` attribute
- No space callbacks initially (spaces not yet supported in claudony)
- No correction/retraction callbacks initially

### Navigation

New "Chat" tab in the fleet home (`app.ts`), alongside Sessions, Cases, Inbox,
Fleet, Mesh. Renders `<claudony-chat-panel>` when active.

Optional `namespace` attribute for case-scoped filtering — set when navigating
from a case context to show only `case-{caseId}/*` channels.

### Existing workbench unchanged

`claudony-workbench.ts` (terminal-primary layout) is not modified. It serves a
different purpose — terminal with conversation as a side panel. The chat tab is
for conversation-primary interaction without a terminal.

## Mosaic Changes

### Files added to `casehub-mosaic/components/channel-activity/`

| File | Purpose |
|------|---------|
| `src/conversation-workbench.ts` | LitElement composition root |
| `src/conversation-workbench.test.ts` | Unit tests |
| `src/responsive.ts` | Media query constants (from chat-app) |
| `src/swipe-controller.ts` | Mobile swipe gesture handler (from chat-app) |
| `src/swipe-controller.test.ts` | Unit tests (from chat-app) |

### Exports added to index

```typescript
export { ConversationWorkbenchElement } from './conversation-workbench.js';
export type { ConversationWorkbenchConfig, LayoutState } from './conversation-workbench.js';
```

### Source material

Extracted from `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` with
these changes:

1. All REST calls replaced with config callbacks
2. Auth (`authenticatedFetch`) replaced with `config.fetch`
3. Push URL parameterized via `config.pushUrl`
4. Identity (`getIdentity()`) replaced with `config.currentActorId`
5. Space management conditionally rendered based on callback presence
6. Correction/retraction conditionally rendered based on callback presence
7. Channel filtering applied when `config.channelFilter` is set
8. Layout store injected via `config.layoutStore`
9. `qhorus-task-panel`, `qhorus-correlation-panel`, `qhorus-artifact-panel`
   replaced with mosaic's built-in `channel-task-panel`, `channel-correlation-panel`,
   `channel-artifact-panel`
10. `chat-demo-identity` removed (app-specific dev tool, not part of the workbench)

## Scope

**In scope (this branch):**
1. `<conversation-workbench>` in casehub-mosaic channel-activity
2. Mosaic SNAPSHOT publish
3. `claudony-chat-panel.ts` wired as fleet home tab
4. Vitest unit tests for claudony-chat-panel
5. E2E test for Chat tab visibility and basic rendering

**Out of scope (separate issues):**
- Chat-app migration to `<conversation-workbench>`
- Deep-link support (`/app/chat/{channelName}`)
- Space management in claudony
- Case context header in chat view

## Testing

### Mosaic (vitest)
- Config with all callbacks → full UI rendered
- Config with no optional callbacks → space/correction UI hidden
- channelFilter applied → only matching channels shown
- sendMessage callback invoked on message post
- layoutStore.save called on dock toggle / panel resize

### Claudony (vitest)
- `claudony-chat-panel` renders `<conversation-workbench>` with correct config
- namespace attribute applies channel filter

### Claudony (E2E)
- Fleet home shows Chat tab
- Chat tab renders workbench with channel list

## References

- `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` — source for extraction
- `claudony/app/src/main/webui/src/components/claudony-workbench.ts` — terminal-primary variant
- `docs/CHANNELS.md` — channel namespace conventions
- casehubio/claudony#287 — conversation polish (component-level improvements)
- casehubio/claudony#158 — debate channel integration
