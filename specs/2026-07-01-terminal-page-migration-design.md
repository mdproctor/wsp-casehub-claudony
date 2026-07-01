# Terminal Page Migration — TypeScript Components via Quinoa

**Date:** 2026-07-01
**Issue:** casehubio/claudony#161
**Status:** Approved

## Context

The terminal page (`session.html` + `terminal.js`) is 834 lines of hand-coded JavaScript serving five concerns: xterm.js terminal, channel panel, worker panel, compose overlay, and iPad key bar. The casehub-pages Quinoa pipeline is already wired (session-grid component migrated). `@casehubio/pages-component-terminal` is in `package.json`. This spec covers migrating the terminal page to TypeScript Web Components composed via `loadSite()`.

## Decision: Separate Entry Point

The terminal page stays as a separate HTML page (`session.html`) with its own esbuild entry point (`terminal.ts` → `dist/terminal.js`). It is NOT folded into the dashboard's single-page app.

**Rationale:**
- Immersive full-viewport layout — xterm.js needs precise pixel dimensions, navigation chrome steals space
- Independent lifecycle — WebSocket + multiple SSE connections live and die with the page
- Bundle splitting — dashboard doesn't need xterm.js; terminal doesn't need session-grid
- Clean URL structure — `?id=SESSION_ID&name=NAME` is bookmarkable

## Component Architecture

Five components, one entry point, one page-level utility:

| Component | Element Name | Responsibility | Reusable? |
|-----------|-------------|---------------|-----------|
| `ClaudonyTerminalHeader` | `claudony-terminal-header` | Back link, session name, status badge, toggle buttons | No |
| `ClaudonyTerminalWorkspace` | `claudony-terminal-workspace` | Three-column flex layout, collapse/expand, event routing | No |
| `ClaudonyWorkerPanel` | `claudony-worker-panel` | Worker list, SSE case-events, click-to-switch | Yes (#75) |
| `ClaudonyChannelPanel` | `claudony-channel-panel` | Channel select, message feed, SSE/polling, stale cursor, case context, lineage | Yes (#75) |
| `ClaudonyKeyBar` | `claudony-key-bar` | Touch device special key buttons | No |
| Compose overlay | (DOM created by entry point) | Modal dialog, Ctrl+G, Ctrl+Enter send | Utility |

### Layout

`ClaudonyTerminalWorkspace` owns the three-column CSS flexbox:
- Left: `ClaudonyWorkerPanel` (240px, collapsible to 0)
- Center: `PagesTerminal` (flex: 1)
- Right: `ClaudonyChannelPanel` (320px, collapsible to 0)

Composed via `loadSite()`:

```typescript
rows(
  hostPanel("terminal-header"),
  hostPanel("terminal-workspace"),
  hostPanel("key-bar"),
)
```

## Event-Driven Coordination

### `pages-event` protocol

All components communicate via the standard `pages-event` custom event defined by casehub-pages. Each component dispatches:

```typescript
this.dispatchEvent(new CustomEvent("pages-event", {
  bubbles: true, composed: true,
  detail: { topic: "terminal-connected", payload: {} },
}));
```

Listeners match on `detail.topic`:

```typescript
this.addEventListener("pages-event", (e: CustomEvent) => {
  const { topic, payload } = e.detail;
  if (topic === "terminal-connected") { /* ... */ }
});
```

The event table below lists `topic` values, not standalone DOM event names.

### Events emitted (bubble up)

| Component | Topic | Payload | Consumer |
|-----------|-------|---------|----------|
| `PagesTerminal` | `terminal-connected` | `{}` | Header → badge "connected" |
| `PagesTerminal` | `terminal-disconnected` | `{ reason }` | Header → badge update |
| `PagesTerminal` | `terminal-resize` | `{ cols, rows }` | Workspace → resize API call |
| `PagesTerminal` | `terminal-ready` | `{ cols, rows }` | Workspace → initial dimensions |
| `ClaudonyWorkerPanel` | `worker-selected` | `{ sessionId, name }` | Workspace → reconfigure terminal + channel panel |
| `ClaudonyKeyBar` | `key-pressed` | `{ code }` | Workspace → `terminal.sendInput(code)` |
| `ClaudonyChannelPanel` | *(none)* | — | Channel panel is event-silent: receives configuration but emits no events upward. All interactions (channel select, message send, SSE connections) are self-contained. |

### Configuration pushed down

| Target | Method | When |
|--------|--------|------|
| `PagesTerminal` | `configure({ wsUrl })` | Mount + worker switch (wsUrl uses `{cols}/{rows}` placeholders — PagesTerminal substitutes at connect time) |
| `ClaudonyWorkerPanel` | `configure({ sessionId })` | Mount |
| `ClaudonyChannelPanel` | `configure({ sessionId, caseId, roleName, createdAt })` | Mount + worker switch |
| `ClaudonyTerminalHeader` | `configure({ sessionName, sessionId })` | Mount |
| `ClaudonyTerminalHeader` | `updateStatus(text, cssClass)` | On terminal events |

### Workspace configuration

Entry point reads URL params (`id`, `name`, `proxyPeer`, `channel`) and fetches `/api/sessions/{id}` once on load. The response provides `caseId`, `roleName`, `status`, `createdAt`. Entry point then calls `configure()` on workspace, passing `proxyPeer` from URL params. Workspace constructs the correct WebSocket URL and resize URL based on whether `proxyPeer` is present:

- Normal: `/ws/{id}/{cols}/{rows}` and `/api/sessions/{id}/resize?cols=X&rows=Y`
- Proxy: `/ws/proxy/{proxyPeer}/{id}/{cols}/{rows}` and `/api/peers/{proxyPeer}/sessions/{id}/resize?cols=X&rows=Y`

### Worker switch flow

1. `ClaudonyWorkerPanel` emits `worker-selected` with `{ sessionId, name }`
2. Workspace updates URL via `history.replaceState`
3. Workspace calls `terminal.configure({ wsUrl: newUrl })` — PagesTerminal tears down + reconnects
4. Workspace calls `channelPanel.configure(...)` — channel panel resets cursors, reconnects SSE
5. Workspace calls `header.configure(...)` and updates worker panel active highlight

### Compose overlay

Entry point creates overlay DOM, listens for Ctrl+G globally, calls `terminal.paste(text)` to send composed text. `paste()` delegates to xterm.js's `Terminal.paste()`, which wraps text in bracketed paste markers (`\x1b[200~...\x1b[201~`) when the terminal has negotiated bracketed paste mode — this prevents shells like zsh from executing each line immediately on multi-line input.

**Upstream change required:** `PagesTerminal` currently only exposes `sendInput(text)` which sends raw bytes. A `paste(text: string)` method must be added that delegates to `this._terminal.paste(text)`. This is tracked as a casehub-pages change.

After `loadSite()` renders the component tree, the entry point obtains the PagesTerminal reference via DOM query: `document.querySelector("pages-component-terminal")`. This reference is used by compose and stored for the `window._xtermTerminal` test hook.

Not a registered panel — page-level utility.

## File Structure

```
src/
├── app.ts                            # dashboard entry (existing, renamed from index.ts)
├── terminal.ts                       # terminal entry (NEW, ~120 lines — loadSite + session fetch + compose overlay)
├── theme.ts                          # shared (existing)
├── util/
│   ├── auth.ts                       # shared (existing)
│   └── time.ts                       # shared (existing)
└── components/
    ├── session-grid.ts               # existing
    ├── terminal-header.ts            # NEW
    ├── terminal-workspace.ts         # NEW
    ├── worker-panel.ts               # NEW
    ├── channel-panel.ts              # NEW (~350 lines — largest component)
    └── key-bar.ts                    # NEW
```

### esbuild config

Two entry points, `outdir` replaces `outfile`, `splitting` eliminates shared code duplication:

```javascript
entryPoints: ["src/app.ts", "src/terminal.ts"],
outdir: "dist",
splitting: true,
format: "esm",
```

Produces `dist/app.js`, `dist/terminal.js`, and shared chunks for common dependencies (pages-runtime, pages-ui, theme, auth, time). The dashboard's existing `<script type="module" src="app.js">` reference is preserved because the entry point is named `app.ts`. xterm.js (~500KB) only appears in `terminal.js` — the dashboard bundle is unaffected.

### session.html

Minimal shell — all structure moves into components. Retains shared stylesheet, xterm CSS, PWA manifest, and service worker registration:

```html
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#1e1e1e">
  <title>RemoteCC Terminal</title>
  <link rel="manifest" href="/manifest.json">
  <link rel="stylesheet" href="/app/style.css">
  <link rel="stylesheet" href="/app/terminal.css">
</head>
<body class="terminal-page">
  <div id="app"></div>
  <script type="module" src="/app/terminal.js"></script>
  <script>
  if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/sw.js');
  }
  </script>
</body>
```

**CSS loading:** `terminal.ts` imports `@casehubio/pages-component-terminal/xterm.css`, which esbuild extracts into `dist/terminal.css`. The `<link>` in session.html loads this bundled CSS. The CDN link is removed — xterm.css is now bundled locally.

## CSS Strategy

Components use **light DOM** (`this.innerHTML`), not shadow DOM. Styles move from `style.css` into each component file as inline `<style>` tags injected into the component's DOM.

Light DOM preserves existing Playwright selectors (`#ch-select`, `.case-worker-row`, etc.) without requiring shadow-piercing selector rewrites.

**Note:** The existing `session-grid.ts` uses Shadow DOM. This inconsistency is intentional for the migration — Light DOM is the architectural direction for all Claudony components. Session-grid will be migrated to Light DOM as part of the dashboard migration (#75).

### What moves where

- **Global styles stay in `style.css`:** `:root` CSS variables (`--bg`, `--surface`, `--border`, `--accent`, etc.), `*` box-sizing reset, `body` background/color, shared `.badge` styles. Both pages (dashboard and terminal) load `style.css`.
- **Terminal-specific CSS moves into components:** ~500 lines of terminal/channel/worker/compose/keybar CSS move out of `style.css` into their respective component files.
- **Dashboard styles stay in `style.css`:** the remaining ~490 lines (session grid, fleet panel, mesh panel, dialogs).
- **xterm.css:** `terminal.ts` imports `@casehubio/pages-component-terminal/xterm.css`. esbuild bundles this (including the transitive `@xterm/xterm/css/xterm.css`) into `dist/terminal.css`. The CDN `<link>` is replaced by a local `<link rel="stylesheet" href="/app/terminal.css">`.

## Testing

### Correctness contract

All 32 existing Playwright E2E tests must pass without changing test assertions:
- `TerminalPageE2ETest` (2 tests)
- `ChannelPanelE2ETest` (19 tests)
- `CaseWorkerPanelE2ETest` (7 tests)
- `CaseContextPanelE2ETest` (4 tests)

### Test mode hooks

Preserve `window.__CLAUDONY_TEST_MODE__` gates:
- `window._xtermTerminal` — exposed from entry point after PagesTerminal configured, via `pagesTerminalElement.terminal` public getter
- `window._caseEventSource` — exposed from worker panel

**Upstream change required:** `PagesTerminal._terminal` is private with no accessor. A public `get terminal(): Terminal | undefined` getter must be added to `PagesTerminal` so the entry point can expose the raw xterm.js Terminal for E2E resize testing (see blog entry mdp02 for rationale).

### Quinoa in test mode

Garden entry GE-20260701-c000c7: Quinoa is disabled in `@QuarkusTest` (hardcoded `LaunchMode.TEST` skip). E2E tests run in dev mode where Quinoa is active. No special handling needed.

## Page lifecycle

### `beforeunload` cleanup

The entry point registers a `beforeunload` handler that tears down resources not covered by `disconnectedCallback` (which does NOT fire on page unload in most browsers):

- Calls `workspace.destroy()` which propagates cleanup to child components
- Worker panel closes its SSE `EventSource`
- Channel panel closes its SSE `EventSource` and clears poll timers
- Entry point clears lineage poll timer and elapsed ticker

This matches the current terminal.js `beforeunload` handler (line 829-833).

## What Does NOT Change

- `session.html` URL and query params (`?id=X&name=Y&proxyPeer=Z&channel=C`)
- REST API endpoints
- WebSocket URL pattern (`/ws/{id}/{cols}/{rows}`)
- E2E test files
- `index.html` / dashboard Quinoa entry (migrated separately — `index.ts` is renamed to `app.ts` to preserve `app.js` output name)

## Upstream changes required (casehub-pages)

These changes to `@casehubio/pages-component-terminal` are prerequisites:

1. **`paste(text: string)` method** — delegates to `this._terminal.paste(text)` for bracketed paste mode support
2. **`get terminal(): Terminal | undefined` getter** — exposes the raw xterm.js Terminal for E2E test hooks

Both are additive, non-breaking changes to the component's public API.

## Migration Approach

Incremental, not big-bang:

1. Land upstream changes to `@casehubio/pages-component-terminal` (paste + terminal getter)
2. Create components and `terminal.ts` entry point
3. Update `esbuild.config.mjs` for two entry points (`app.ts`, `terminal.ts`) with `splitting: true`
4. Remove old `META-INF/resources/app/terminal.js` and update `session.html` to load new bundle (same step — eliminates resource path collision between Quinoa output and static file; old file is recoverable from git)
5. Move CSS from `style.css` into components
6. Run E2E tests — all 32 must pass
