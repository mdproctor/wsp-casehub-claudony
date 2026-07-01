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

### Events emitted (bubble up)

| Component | Event | Payload | Consumer |
|-----------|-------|---------|----------|
| `PagesTerminal` | `terminal-connected` | `{}` | Header → badge "connected" |
| `PagesTerminal` | `terminal-disconnected` | `{ reason }` | Header → badge update |
| `PagesTerminal` | `terminal-resize` | `{ cols, rows }` | Workspace → resize API call |
| `PagesTerminal` | `terminal-ready` | `{ cols, rows }` | Workspace → initial dimensions |
| `ClaudonyWorkerPanel` | `worker-selected` | `{ sessionId, name }` | Workspace → reconfigure terminal + channel panel |
| `ClaudonyKeyBar` | `key-pressed` | `{ code }` | Workspace → `terminal.sendInput(code)` |

### Configuration pushed down

| Target | Method | When |
|--------|--------|------|
| `PagesTerminal` | `configure({ wsUrl })` | Mount + worker switch |
| `ClaudonyWorkerPanel` | `configure({ sessionId })` | Mount |
| `ClaudonyChannelPanel` | `configure({ sessionId, caseId, roleName, createdAt })` | Mount + worker switch |
| `ClaudonyTerminalHeader` | `configure({ sessionName, sessionId })` | Mount |
| `ClaudonyTerminalHeader` | `updateStatus(text, cssClass)` | On terminal events |

### Session fetch

Entry point fetches `/api/sessions/{id}` once on load. The response provides `caseId`, `roleName`, `status`, `createdAt`. Entry point then calls `configure()` on all components.

### Worker switch flow

1. `ClaudonyWorkerPanel` emits `worker-selected` with `{ sessionId, name }`
2. Workspace updates URL via `history.replaceState`
3. Workspace calls `terminal.configure({ wsUrl: newUrl })` — PagesTerminal tears down + reconnects
4. Workspace calls `channelPanel.configure(...)` — channel panel resets cursors, reconnects SSE
5. Workspace calls `header.configure(...)` and updates worker panel active highlight

### Compose overlay

Entry point creates overlay DOM, listens for Ctrl+G globally, calls `terminal.sendInput(text)` directly (method already exists on `PagesTerminal`). Not a registered panel — page-level utility.

## File Structure

```
src/
├── index.ts                          # dashboard entry (existing)
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

Two entry points, `outdir` replaces `outfile`:

```javascript
entryPoints: ["src/index.ts", "src/terminal.ts"],
outdir: "dist",
```

Produces `dist/index.js` and `dist/terminal.js`.

### session.html

Minimal shell — all structure moves into components:

```html
<body class="terminal-page">
  <div id="app"></div>
  <script type="module" src="/app/terminal.js"></script>
</body>
```

## CSS Strategy

Components use **light DOM** (`this.innerHTML`), not shadow DOM. Styles move from `style.css` into each component file as inline `<style>` tags injected into the component's DOM.

Light DOM preserves existing Playwright selectors (`#ch-select`, `.case-worker-row`, etc.) without requiring shadow-piercing selector rewrites.

The ~500 lines of terminal/channel/worker/compose/keybar CSS move out of `style.css` into their respective components. The remaining ~490 lines (dashboard styles) stay in `style.css`.

## Testing

### Correctness contract

All 32 existing Playwright E2E tests must pass without changing test assertions:
- `TerminalPageE2ETest` (2 tests)
- `ChannelPanelE2ETest` (19 tests)
- `CaseWorkerPanelE2ETest` (7 tests)
- `CaseContextPanelE2ETest` (4 tests)

### Test mode hooks

Preserve `window.__CLAUDONY_TEST_MODE__` gates:
- `window._xtermTerminal` — exposed from workspace after PagesTerminal configured
- `window._caseEventSource` — exposed from worker panel

### Quinoa in test mode

Garden entry GE-20260701-c000c7: Quinoa is disabled in `@QuarkusTest` (hardcoded `LaunchMode.TEST` skip). E2E tests run in dev mode where Quinoa is active. No special handling needed.

## What Does NOT Change

- `session.html` URL and query params (`?id=X&name=Y&proxyPeer=Z&channel=C`)
- REST API endpoints
- WebSocket URL pattern (`/ws/{id}/{cols}/{rows}`)
- E2E test files
- `dashboard.js` / `index.html` (migrated separately)

## Migration Approach

Incremental, not big-bang:

1. Create components and `terminal.ts` entry point
2. Update `esbuild.config.mjs` for two entry points
3. Update `session.html` to load new bundle
4. Move CSS from `style.css` into components
5. Run E2E tests — all 32 must pass
6. Remove old `terminal.js` only after E2E green
