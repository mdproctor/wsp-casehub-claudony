# Adopt casehub-pages for UI Composition via Quinoa

**Issue:** casehubio/claudony#161
**Date:** 2026-06-30
**Status:** Approved
**Depends on:** None — all required primitives (`stack`, `split`, `hostPanel`, `registerPanel`, `activateSlot`) already exist in casehub-pages. `pages-event` is designed and partially implemented (dispatched by `websocket-source.ts`, documented in workbench-primitives spec, tested) but has no central runtime handler — this spec uses it as a plain DOM event bus, which requires no runtime support. casehub-pages#64 tracks additional workbench features (dockable panels, topbar/status bar) that this spec does not use.
**Cross-repo deliverable:** casehubio/casehub-pages — `@casehubio/pages-component-terminal` (new stock component)

## Summary

Replace Claudony's hand-coded vanilla JS frontend (~2600 lines across `dashboard.js`, `terminal.js`, `style.css`) with a casehub-pages application. Pages provides the workbench shell, layout, data pipeline, and inter-component communication. Claudony provides custom panels as Web Components. The terminal becomes a stock pages component reusable across the platform.

## Architecture

### Approach: Hybrid (C → A convergence)

Use the pages data pipeline where it adds value today (session list via WebSocket source, fleet via REST polling). Keep custom data connections inside components where pages doesn't yet have the transport (SSE for channels, case workers, mesh — waiting on casehub-pages#74). Each custom connection migrates to the pages pipeline when the SSE source lands — the swap is internal to the component, the external API doesn't change.

### Application Structure

Root is a `stack()` (non-lazy, the default) with two views: a **selector** (session grid + fleet + mesh) and a **workbench** (terminal + channels + case workers). Non-lazy mode keeps inactive slots mounted with `display: none` — the terminal WebSocket stays connected when navigating back to the selector, avoiding reconnect and history replay cost on switch-back. Components handle cleanup via `session-deselected` event handlers, not `disconnectedCallback()`.

Session selection in the grid fires a `pages-event` that triggers `activateSlot` to switch the stack to the workbench view, passing the session context to all workbench panels.

```typescript
import { stack, split, rows, hostPanel, dataset } from "@casehubio/pages-ui";
import { loadSite, registerPanel } from "@casehubio/pages-runtime";

registerPanel("session-grid", "claudony-session-grid");
registerPanel("fleet",        "claudony-fleet");
registerPanel("mesh",         "claudony-mesh");
registerPanel("terminal",     "pages-terminal");
registerPanel("channels",     "claudony-channels");
registerPanel("case-workers", "claudony-case-workers");

const selector = rows(
  split("horizontal", [
    hostPanel("fleet"),
    hostPanel("session-grid"),
    hostPanel("mesh"),
  ], { ratio: [15, 60, 25] }),
);

const workbench = split("horizontal", [
  hostPanel("case-workers"),
  hostPanel("terminal"),
  hostPanel("channels"),
], { ratio: [15, 60, 25] });

const app = stack(selector, workbench);

const site = await loadSite(document.getElementById("app")!, app);
```

## Web Components

Six components with clear responsibility boundaries:

| Component | Location | Data Source | Pages Integration |
|-----------|----------|-------------|-------------------|
| `claudony-session-grid` | Claudony webui | Pages WebSocket dataset | `pages-event` emitter (selection → `activateSlot` stack switch) |
| `claudony-fleet` | Claudony webui | Pages REST dataset (`refreshTime: "10s"`) | Receives session context; peer CRUD (add, remove, ping, mode toggle) |
| `claudony-mesh` | Claudony webui | Custom SSE/polling → pages pipeline when #74 lands | Event bus for channel selection |
| `pages-terminal` | casehub-pages repo | Own bidirectional WebSocket (I/O, not data) | Listens for `session-selected` `pages-event`; initial config via `configure()` |
| `claudony-channels` | Claudony webui | Custom SSE/polling → pages pipeline when #74 lands | Listens for `session-selected` + `worker-selected` `pages-event` |
| `claudony-case-workers` | Claudony webui | Custom SSE → pages pipeline when #74 lands | `pages-event` emitter (worker click → terminal reconnects) |

**Component contract:** standard `HTMLElement` subclass with Shadow DOM. Pages mounts via `hostPanel()`, calls `configure(props)` before `appendChild()` (one-time — props are not reactive). Components own internal DOM and any custom data connections. Outward communication through `pages-event` custom events (bubbling, composed). Inbound context changes (e.g., session selection) are received by listening for `pages-event` on `document` — not via prop updates.

**Compose overlay and mobile key bar:** Claudony-specific UI rendered as siblings to the `<div id="app">` root, outside the pages layout tree. These use `position: fixed` and are controlled by Claudony's own keyboard handlers (Ctrl+G for compose). They are not part of any pages component or layout slot.

**C → A migration path:** when casehub-pages#74 (SSE source) lands, `claudony-mesh`, `claudony-channels`, and `claudony-case-workers` swap internal fetch/SSE code for a pages dataset binding. Component boundary and external API unchanged — only internal data plumbing changes.

## Data Flow

### Session list — pages WebSocket data source

```typescript
const wsBase = `${location.protocol === "https:" ? "wss:" : "ws:"}//${location.host}`;
dataset("sessions", `${wsBase}/ws/sessions-feed`);
```

Pages detects WebSocket sources by URL scheme (`ws://` or `wss://`), not by a configuration property. The runtime constructs the full URL from the current page origin.

New server-side WebSocket endpoint (`/ws/sessions-feed`) speaks the pages wire protocol: `snapshot` on connect, then `replace`/`append`/`remove` events as sessions change. Existing REST endpoint `/api/sessions` unchanged for non-pages consumers.

**Implementation:** New class `SessionFeedWebSocket` in `claudony-app`, annotated `@WebSocket(path = "/ws/sessions-feed")`. On connect: sends a `snapshot` message with all sessions from `SessionRegistry.all()`. Authentication follows the same implicit tenant-scoping as `TerminalWebSocket` — `SessionRegistry.all()` filters by `TenantContext.currentTenantId()`.

**SessionRegistry change notification rework:** The existing `addChangeListener(Consumer<String>)` is inadequate for session feeds — it passes `caseId` (not `sessionId`), skips sessions without a `caseId`, doesn't fire on `register()` or `touch()`, and fires `remove()` notifications before the session is actually removed. Replace with a typed session event API:

```java
public record SessionEvent(String sessionId, Type type) {
    enum Type { ADDED, UPDATED, REMOVED }
}
void addSessionListener(Consumer<SessionEvent> listener);
```

Fire from `register()` (ADDED), `updateStatus()` (UPDATED), `touch()` (UPDATED, throttled — lastActive changes on every interaction), and `remove()` (REMOVED, fired after `sessions.remove()`). The existing `caseId`-based listener stays for its current consumers (case-level notifications); the new API is session-level.

**Snapshot-subscribe atomicity:** The listener is registered before the snapshot is sent. Any change events arriving between registration and snapshot delivery are queued and flushed after the snapshot. This avoids the history-replay race documented in ARC42STORIES §6.

**Federation:** Initial implementation covers local sessions only. Federated peer sessions are deferred — the existing REST endpoint `/api/peers` handles fleet visibility. When federation is needed, `SessionFeedWebSocket` can subscribe to peer change events via the existing mesh protocol.

### Fleet — pages REST polling

```typescript
dataset("fleet", "/api/peers", { refreshTime: "10s" });
```

No new endpoint — pages polls the existing REST API.

### Session selection propagation

1. `claudony-session-grid` dispatches `pages-event` with `{ topic: "session-selected", payload: { sessionId, wsUrl } }`
2. App-level listener calls `activateSlot(stackEl, "workbench")` — switches to workbench view
3. All workbench components listen for `pages-event` topic `session-selected` on `document`
4. `pages-terminal` closes any existing WebSocket and opens a new one to `wss://host/ws/{sessionId}`
5. `claudony-channels` and `claudony-case-workers` start SSE connections scoped to that session
6. Worker click in `claudony-case-workers` dispatches `pages-event` with `{ topic: "worker-selected", payload: { sessionId, workerId } }` — terminal and channels reconnect to the new session context

### Back navigation

A "back to sessions" button dispatches `pages-event` with `{ topic: "session-deselected" }`. App-level listener calls `activateSlot(stackEl, "selector")`. Terminal WebSocket disconnects, SSE connections close — components handle cleanup in their `session-deselected` event handler (not via unmount, since non-lazy `stack()` keeps inactive slots mounted).

### URL state management

The app preserves view state in the URL so that page refresh, deep links, and browser back/forward work:

| State | URL parameter | Example |
|-------|---------------|---------|
| Workbench view active | `?view=workbench` | `/?view=workbench&session=abc123` |
| Selected session | `&session=<id>` | Deep link to a session |
| Selected worker | `&worker=<id>` | Refresh returns to the same worker |

**Write:** The app-level `pages-event` listeners call `history.replaceState()` on `session-selected`, `worker-selected`, and `session-deselected` events — same pattern as the current `history.replaceState()` in `switchToWorker()`.

**Read:** On initial load, `index.ts` reads URL search params. If `?view=workbench&session=<id>` is present, it dispatches a synthetic `session-selected` event to restore state — triggering `activateSlot` and component initialization as if the user had clicked the session.

**Browser back:** `popstate` event listener dispatches the appropriate `pages-event` to sync view state with the URL.

### `pages-event` contract

Components communicate via the standard DOM `pages-event` custom event. No pages-runtime handler is required — the event is a plain DOM event bus.

```typescript
interface PagesEventDetail {
  topic: string;
  payload?: Record<string, unknown>;
}

// Dispatch (from any component)
this.dispatchEvent(new CustomEvent("pages-event", {
  bubbles: true,
  composed: true,
  detail: { topic: "session-selected", payload: { sessionId, wsUrl } },
}));

// Listen (on document — events bubble from any shadow root)
document.addEventListener("pages-event", (e: CustomEvent<PagesEventDetail>) => {
  if (e.detail.topic === "session-selected") { /* ... */ }
});
```

Topics used by this app: `session-selected`, `session-deselected`, `worker-selected`.

`activateSlot` is imported from `@casehubio/pages-component` (not `pages-runtime`):

```typescript
import { activateSlot } from "@casehubio/pages-component";
```

## Terminal Component (`pages-terminal`)

Stock casehub-pages component at `@casehubio/pages-component-terminal`. Any host app can mount it.

### Props

```typescript
interface TerminalProps {
  wsUrl: string;
  fontSize?: number;       // default 14
  fontFamily?: string;     // default "Menlo, Monaco, monospace"
  scrollback?: number;     // default 5000
  fitToContainer?: boolean; // default true
}
```

### Internal responsibilities

- xterm.js instantiation with fit addon
- WebSocket connection (connect, reconnect with exponential backoff)
- History replay on connect
- Input via WebSocket upstream
- Resize observation → `fit()` → sends dimensions to server
- Picks up pages CSS custom properties for theming

### NOT owned by the terminal component

- Compose overlay (Ctrl+G) — Claudony-specific, rendered alongside by Claudony
- Mobile key bar — Claudony-specific, same approach
- Session status badge — pages layout handles (workbench header)

### Packaging

**Direct Web Component** (not iframe-isolated). Unlike `pages-component-llm-prompter` which uses the iframe-based `pages-iframe-api`, the terminal is a direct `HTMLElement` subclass with Shadow DOM. Rationale: xterm.js requires precise resize handling tied to the parent container (`ResizeObserver` → `fit()`), direct keyboard event interception, and low-latency I/O — all of which are degraded by iframe boundaries. Shadow DOM provides sufficient CSS isolation.

**Bundle size:** xterm.js + fit addon is ~300KB. This is acceptable for a stock component that apps opt into — the cost is paid only by apps that mount `pages-terminal`, not by the pages runtime. The component is tree-shaken from apps that don't import it.

Standalone package in `components/pages-component-terminal/`, distributed via `@casehubio/pages-component-terminal`. Mounted as `hostPanel` — opaque to pages, doesn't participate in data pipeline.

## Migration Mapping

### Existing files

| Current file | Fate | Notes |
|---|---|---|
| `app/index.html` | **Replaced** | Minimal shell: `<div id="app">`, loads `app.js` |
| `app/session.html` | **Deleted** | Workbench is a stack view, not a separate page |
| `app/dashboard.js` | **Decomposed** | → `claudony-session-grid`, `claudony-fleet`, `claudony-mesh` (see feature migration table below) |
| `app/terminal.js` | **Decomposed** | → `pages-terminal` (upstream), `claudony-channels`, `claudony-case-workers` + compose overlay and key bar as Claudony root-level UI |
| `app/style.css` | **Split** | Each component owns styles via Shadow DOM. Shell theming via pages CSS custom properties. Shared tokens in `theme.ts` |
| `auth/login.html` | **Stays** | `META-INF/resources/auth/` — orthogonal to pages |
| `auth/register.html` | **Stays** | Same |
| `manifest.json` | **Moves** | → `src/main/webui/public/` |
| `sw.js` | **Moves** | → `src/main/webui/public/` |
| `icons/` | **Moves** | → `src/main/webui/public/icons/` |

### Feature migration table

Every interactive feature in `dashboard.js` and `terminal.js` maps to a specific destination:

| Feature | Current location | Destination |
|---------|-----------------|-------------|
| Session grid / list | `dashboard.js` grid rendering | `claudony-session-grid` component |
| Session creation dialog | `dashboard.js` `submitSession()` | `claudony-session-grid` (create action in grid toolbar) |
| Session deletion | `dashboard.js` delete button handler | `claudony-session-grid` (delete action per row) |
| Session overwrite confirmation | `dashboard.js` `overwriteBtn` | `claudony-session-grid` (confirmation modal internal to component) |
| PR status / git info | `dashboard.js` `prStatusHtml()`, `/api/sessions/{id}/git-status` | `claudony-session-grid` (status column with fetch) |
| Service health check | `dashboard.js` `svc-btn` handler | `claudony-session-grid` (health indicator per row) |
| iTerm2 "Open in iTerm2" | `dashboard.js` `iterm-btn` handler | `claudony-session-grid` (action button per row) |
| Dev-login auth modal | `dashboard.js` `showAuthDialog()` | `util/auth.ts` (401 interceptor + modal, rendered at app root) |
| Peer add/remove/ping/mode toggle | `dashboard.js` peer management section | `claudony-fleet` (full CRUD — not read-only) |
| MeshPanel overview + interjection | `dashboard.js` `MeshPanel._send()` | `claudony-mesh` (human interjection dock inside component) |
| Terminal I/O | `terminal.js` xterm.js + WebSocket | `pages-terminal` stock component |
| Stale cursor prompt | `terminal.js` `showStalePrompt()` | `claudony-channels` (operates on channel cursors and timeline endpoints, not terminal state) |
| Compose overlay (Ctrl+G) | `terminal.js` compose UI | Claudony root-level UI (outside pages layout tree) |
| Mobile key bar | `terminal.js` key bar | Claudony root-level UI (outside pages layout tree) |
| Channel feed + send | `terminal.js` channel panel | `claudony-channels` component |
| Case worker list + switch | `terminal.js` worker panel | `claudony-case-workers` component |

### New file structure

```
src/main/webui/
├── package.json
├── esbuild.config.mjs
├── tsconfig.json
├── .npmrc
├── public/
│   ├── index.html
│   ├── manifest.json
│   ├── sw.js
│   └── icons/
└── src/
    ├── index.ts              — loadSite() entry point, layout tree
    ├── theme.ts              — design tokens, CSS custom properties
    ├── components/
    │   ├── session-grid.ts   — claudony-session-grid
    │   ├── fleet.ts          — claudony-fleet
    │   ├── mesh.ts           — claudony-mesh
    │   ├── channels.ts       — claudony-channels
    │   └── case-workers.ts   — claudony-case-workers
    └── util/
        ├── auth.ts           — 401 handling, dev-login modal
        └── time.ts           — relative time formatting
```

### Server-side changes

- **New endpoint:** `/ws/sessions-feed` — WebSocket speaking pages wire protocol, pushes session state changes
- **Existing endpoints:** unchanged (REST API, terminal WebSocket, SSE streams)
- **`application.properties`:** add `quarkus.quinoa.build-dir=dist`, `quarkus.quinoa.package-manager-install=true`
- **`pom.xml`:** add `quarkus-quinoa` dependency

### Test impact

- **`StaticFilesTest`** — rewrite to verify Quinoa-served paths. Quinoa serves the esbuild `dist/` output. New assertions: `/index.html` (entry point), `/app.js` (bundled JS), `/app.css` (bundled CSS), `/manifest.json`, `/sw.js`, `/icons/icon-192.svg`, `/icons/icon-512.svg`. Content assertions update: `index.html` contains `<div id="app">` and loads `app.js`; previous `dashboard.js`/`terminal.js` references no longer apply.
- **`AppAuthProtectionTest`** — same assertions, paths unchanged (`/auth/*` routes are outside Quinoa)
- **Playwright E2E tests** — Playwright's `page.locator()` auto-pierces open Shadow DOM (built-in since Playwright 1.27). Tests update selectors for new custom element tag names (e.g., `page.locator("claudony-session-grid")`) but do not require shadow-piercing workarounds. ID-based selectors inside shadow roots work via Playwright's default CSS engine.
- All test categories (dashboard, terminal, channel, case worker, mesh) map 1:1 to new components

### `.npmrc` configuration

```
@casehubio:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

CI authenticates via `GITHUB_TOKEN` environment variable (already available in GitHub Actions). Local dev: developers set `GITHUB_TOKEN` in their shell profile or use `gh auth token` to populate it. This is the standard casehub-pages convention documented in the Quinoa convention guide.

## Cross-Repo Issues to File

| Repo | Issue | Description |
|------|-------|-------------|
| casehub-pages | `pages-component-terminal` | Stock xterm.js terminal component — `hostPanel` mounted, props-driven, reusable across platform |
| casehub-pages | SSE source (#74 exists) | Confirm Claudony channel/mesh/case-worker panels as consumers |

## Garden Context

- **GE-20260629-59c7e6** — esbuild minifies TypeScript constants inside template literal `html()` strings. Relevant when writing component templates — avoid interpolating constants inside `html()` template literals that contain `<script>` blocks.
- **GE-20260429-07114f** — PlaywrightBase BASE_URL / random-port. Existing E2E test infrastructure; no change needed.

## Implementation Phasing

This spec describes the full architectural vision. Issue #161 covers phase 1 only. Subsequent phases will be tracked as child issues of #161 (or a new parent epic):

| Phase | Scope | Issue |
|-------|-------|-------|
| 1 | Quinoa setup + esbuild + `claudony-session-grid` rendering via `loadSite()` | #161 |
| 2 | `pages-terminal` stock component in casehub-pages | File on casehub-pages |
| 3 | Remaining Claudony components (`fleet`, `mesh`, `channels`, `case-workers`) | File on claudony |
| 4 | `/ws/sessions-feed` server-side endpoint + WebSocket dataset wiring | File on claudony |
| 5 | Delete old `dashboard.js`, `terminal.js`, `session.html` | File on claudony |

Issue #161's scale should be updated from S to M and complexity from Low to Med to reflect phase 1 scope (Quinoa setup + first component migration).

## Deferred

- Compose overlay and mobile key bar as pages primitives — evaluate once workbench is running
- Drag-and-drop panel rearrangement (casehub-pages#75) — future epic, not needed for initial adoption
- WebSocket data source for channels/mesh/case-workers — migrates when casehub-pages#74 lands
- Federated session feed (peer sessions in `/ws/sessions-feed`) — deferred until fleet federation needs real-time updates
