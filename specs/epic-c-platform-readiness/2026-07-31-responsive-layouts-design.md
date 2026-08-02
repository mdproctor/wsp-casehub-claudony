# Responsive Layouts Design

**Issue:** casehubio/claudony#179
**Date:** 2026-07-31
**Status:** Approved

## Context

The workbench has two distinct component variants with no responsive breakpoints:

- **`claudony-workbench`** — used for case-bound sessions (with `caseId`). Renders a conditional worker nav panel + terminal + conversation area (with channel-nav, feed, input, dock strip) + conditional context panel, all within a single shadow root.
- **`terminal-workspace`** — used for standalone sessions (no `caseId`). Renders three sibling custom elements: `<claudony-worker-panel>` + terminal container + `<claudony-channel-panel>`, each with their own shadow DOM.

The session page entry point (`terminal.ts`) dynamically replaces `terminal-workspace` with `claudony-workbench` at runtime when a case-bound session is detected.

Desktop is the primary target; tablets and phones need graceful degradation. The fleet home page (`session-grid`, `style.css`) has existing `@media (max-width: 600px)` rules (fleet panel hidden, single-column grid); the workbench components have none.

The chat-app (standalone Qhorus demo) already handles responsive layouts across desktop/tablet/phone. We take inspiration from its breakpoints and navigation patterns but implement fresh in the workbench — the layouts are structurally different (terminal + conversation + context vs channel list + feed).

## Design Decisions

- **Progressive degradation** with three tiers: desktop, tablet, phone.
- **Context-dependent tablet layout** — case-bound sessions prioritise conversation; standalone sessions prioritise terminal.
- **Conversation-first phone layout** — channel feed + input are the default view; terminal is a secondary tab.
- **CSS media queries + minimal JS state** — media queries in Shadow DOM set the layout structure at each breakpoint. On the phone tier, a reactive `_activeTab` Lit state property manages full-screen panel switching. No `matchMedia()` or `ResizeObserver` for tier detection — the CSS media queries are the sole mechanism for selecting the layout tier.
- **No swipe gestures initially** — tab switching is simpler and more discoverable. Swipe can be added later.
- **Both workbench components adapt independently** — `claudony-workbench` and `terminal-workspace` each implement their own responsive CSS within their shadow roots; they are structurally different components requiring different responsive treatments.
- **Breakpoint migration** — existing `@media (max-width: 600px)` rules in `style.css` and `claudony-fleet-panel.ts` are migrated to `768px` to align with the new phone breakpoint, eliminating the dead zone between 601px-767px.

## Breakpoints

| Tier | Range | Devices |
|------|-------|---------|
| Desktop | >1024px | Laptops, monitors |
| Tablet | 768-1024px | iPad portrait, Android tablets |
| Phone | <768px | iPhone, Android phones |

**Migration:** Existing `@media (max-width: 600px)` rules in `style.css` (session grid, header padding, fleet panel hiding) and `claudony-fleet-panel.ts` Shadow DOM are migrated to `768px`.

## Desktop (>1024px) — No Change

Current layout preserved. The nav panel and context panel are conditional, not persistent columns.

### `claudony-workbench` (case-bound sessions)

```
+--------+------------------+-------------+-----------+
| Nav    | Terminal         | Conversation| Context   |
| 220px  | flex: 1          | 360px       | 280px     |
| (cond) |                  |             | (cond)    |
+--------+------------------+-------------+-----------+
  ^                                          ^
  Only when workers > 0                     Only when dock button active
```

- Nav panel: 220px worker list, rendered only when `_workers.length > 0`
- Main panel: terminal (flex: 1)
- Conversation area: channel-nav + channel-feed + channel-input + dock strip (360px, always present)
- Context panel: task/correlation/artifact/member panels (280px, visible only when a dock button is active)

### `terminal-workspace` (standalone sessions)

```
+-----------+------------------+------------+
| Workers   | Terminal         | Channel    |
| 240px     | flex: 1          | 300px      |
| (collapsed|                  | (collapsed |
|  default) |                  |  default)  |
+-----------+------------------+------------+
  Both panels start collapsed; toggled via header buttons
```

- Worker panel: `<claudony-worker-panel>` (240px, collapsed by default via `.collapsed` class)
- Terminal container: flex: 1
- Channel panel: `<claudony-channel-panel>` (300px, collapsed by default)

## Tablet (768-1024px)

### Case-bound sessions → `claudony-workbench`

Conversation stays visible. Terminal shares space. Nav collapses.

```
+--+------------------+-------------+
|  | Terminal         | Conversation|
|  | flex: 1          | 320px       |
|  |                  |             |
+--+------------------+-------------+
 ^
 Icon rail (48px) — expand to overlay drawer on tap
```

CSS changes in `claudony-workbench.ts` Shadow DOM:
- `.nav-panel`: width 220px → 48px, show icons only. Tap handler toggles a `_navDrawerOpen` state that renders the full nav as an overlay (`position: absolute; z-index: 10`) over the terminal area within the same shadow root.
- `.conversation-area`: width 360px → 320px.
- `.context-panel`: hidden. Case header moves into the conversation area header. Dock panels (tasks, correlation, artifacts) open as overlays from the dock strip.

### Standalone sessions → `terminal-workspace`

Terminal takes full width. Conversation available via slide-out.

```
+--+--------------------------------------+
|  | Terminal                              |
|  | flex: 1                               |
|  |                                       |
+--+--------------------------------------+
 ^
 Icon rail (48px)
```

CSS changes in `terminal-workspace.ts` Shadow DOM and `worker-panel.ts` / `channel-panel.ts`:
- Worker panel: collapses to icon rail (48px). Expand via overlay — `:host` positioned with `position: fixed; z-index: 100; left: 0; top: 0; bottom: 0; width: 240px`. `position: fixed` is relative to the viewport, so it works across shadow DOM boundaries even though the worker panel is a sibling custom element.
- Channel panel: hidden by default. A "chat" icon in the icon rail opens it as an overlay drawer — `:host` positioned with `position: fixed; right: 0; top: 0; bottom: 0; width: 300px; z-index: 100`.
- Terminal container: gets remaining space.

## Phone (<768px)

### Panel Visibility Strategy

Full-screen panels switched via bottom tab bar. To avoid destroying the xterm.js instance, non-active panels use layered absolute positioning with `visibility: hidden`:

```css
/* Inside workbench / terminal-workspace shadow DOM */
.tab-content {
  position: relative;
  flex: 1;
  overflow: hidden;
}
.tab-panel {
  position: absolute;
  inset: 0;
  visibility: hidden;
}
.tab-panel.active {
  visibility: visible;
  z-index: 1;
}
```

All panels occupy the same space at full size. `visibility: hidden` keeps the terminal in the DOM with correct dimensions while hidden — no xterm.js reattachment or dimension loss. On tab switch, the workbench sets `_activeTab` state and calls `requestAnimationFrame(() => this._fitTerminal())` as a defensive measure.

### Tab State Management

Both `claudony-workbench` and `terminal-workspace` gain a new reactive property:

```typescript
// claudony-workbench (case-bound): chat is default
@state() private _activeTab: 'chat' | 'terminal' | 'context' = 'chat';

// terminal-workspace (standalone): terminal is default
@state() private _activeTab: 'terminal' | 'chat' = 'terminal';
```

The bottom tab bar is rendered inline in the shadow DOM `render()` method — no separate component. Communication is direct state mutation within the same shadow root.

### Case-bound sessions → `claudony-workbench`

Conversation-first. Full-screen panels switched via bottom tab bar.

```
+----------------------------------+
| Header (compact)                 |
+----------------------------------+
|                                  |
| Channel feed + input             |
| (full screen)                    |
|                                  |
+----------------------------------+
| [Chat] [Terminal] [Context]      |
+----------------------------------+
       bottom tab bar
```

- Channel feed + input fill the viewport between header and tab bar.
- Tab bar: 3 tabs — Chat (default), Terminal, Context.
- Terminal tab: full-screen terminal, input via on-screen keyboard.
- Context tab: case header, lineage, workers list.
- Nav panel: channel switching via a dropdown or sheet triggered from the header (no persistent sidebar).
- Dock panels (tasks, correlation, artifacts): accessible from the Context tab.
- Outer header: hamburger menu (channel picker), truncated session name, compact status badge.

### Standalone sessions → `terminal-workspace`

Terminal-first. Single tab for channel if available.

```
+----------------------------------+
| Header (compact)                 |
+----------------------------------+
|                                  |
| Terminal                          |
| (full screen)                    |
|                                  |
+----------------------------------+
| [Terminal] [Chat]                |
+----------------------------------+
```

- Terminal fills the viewport by default.
- Tab bar: 2 tabs — Terminal (default), Chat.
- Worker panel: hidden entirely on phone (no tab, no access — standalone sessions have no case context).
- Outer header: back link, truncated session name, status badge.

#### Render structure change

The current `terminal-workspace.ts` renders sibling custom elements. On phone, the `render()` method wraps them in `.tab-panel` containers for tab switching:

```html
<!-- terminal-workspace.ts render() at phone breakpoint -->
<div class="tab-content">
  <div class="tab-panel ${this._activeTab === 'terminal' ? 'active' : ''}">
    <div id="terminal-container"></div>
  </div>
  <div class="tab-panel ${this._activeTab === 'chat' ? 'active' : ''}">
    <claudony-channel-panel></claudony-channel-panel>
  </div>
</div>
<nav class="tab-bar" role="tablist" aria-label="Panel navigation">
  <button role="tab" aria-selected=${this._activeTab === 'terminal'}
    @click=${() => this._switchTab('terminal')}>Terminal</button>
  <button role="tab" aria-selected=${this._activeTab === 'chat'}
    @click=${() => this._switchTab('chat')}>Chat</button>
</nav>
```

The worker panel is omitted from the phone render — it has no tab and no access point. The `.tab-content` wrapper uses the same absolute-positioning strategy as `claudony-workbench` (§Panel Visibility Strategy). The channel panel's lifecycle (SSE connections, state) is preserved since it remains in the DOM regardless of tab visibility.

### Tab Bar Specification

The tab bar is rendered inline in each component's `render()` method, visible only at the phone breakpoint via CSS media query:

```html
<nav class="tab-bar" role="tablist" aria-label="Panel navigation">
  <button role="tab" aria-selected=${this._activeTab === 'chat'}
    tabindex=${this._activeTab === 'chat' ? 0 : -1}
    @click=${() => this._switchTab('chat')}>Chat</button>
  <button role="tab" aria-selected=${this._activeTab === 'terminal'}
    tabindex=${this._activeTab === 'terminal' ? 0 : -1}
    @click=${() => this._switchTab('terminal')}>Terminal</button>
  <!-- Context tab: workbench only, when caseId present -->
</nav>
```

- **Location:** Inside the workbench/terminal-workspace shadow DOM. Direct access to component state — no event protocol needed.
- **Accessibility:** `role="tablist"` on container, `role="tab"` + `aria-selected` on buttons, `tabindex` roving for keyboard navigation, arrow keys switch between tabs.
- **Styling:** Fixed to bottom of component within the shadow root. 48px height, equal-width tabs, 44px minimum touch targets. `padding-bottom: env(safe-area-inset-bottom)` ensures the home indicator on iPhone X+ (all models since 2017) doesn't obscure the tab buttons.
- **Visibility:** Hidden at desktop and tablet tiers via `@media (min-width: 768px) { .tab-bar { display: none; } }`.
- **Tab switch handler:** Sets `_activeTab`, dispatches `pages-event` with topic `active-tab-changed` and payload `{ tab }` (for key-bar visibility — see below), then calls `requestAnimationFrame(() => this._fitTerminal())` to ensure xterm.js recalculates dimensions.

### Key-bar Phone Tab Visibility

The key-bar (`<claudony-key-bar>`) is a `hostPanel` in the `rows()` layout, outside the workbench shadow DOM. Its buttons (Esc, Ctrl+C, Tab, arrow keys) dispatch `key-pressed` events that send terminal input. On phone, showing terminal-specific keys while the user is on the Chat or Context tab is confusing and could cause accidental terminal input.

The `_switchTab()` handler dispatches a `pages-event` with topic `active-tab-changed` and payload `{ tab }`. The key-bar listens on the document for this event and hides when `tab !== 'terminal'`:

```typescript
// In key-bar.ts connectedCallback():
document.addEventListener('pages-event', ((e: CustomEvent) => {
  if (e.detail?.topic === 'active-tab-changed') {
    this._terminalActive = e.detail.payload.tab === 'terminal';
  }
}) as EventListener);
```

On desktop and tablet, no `active-tab-changed` event is fired, so the key-bar remains visible as normal (governed only by its existing touch detection). This preserves the pages-runtime composition and uses the established event protocol.

## Terminal Resize Protocol

Layout transitions that change the terminal container dimensions must trigger the resize chain:

1. **Layout change** (CSS media query breakpoint crossed, or phone tab switch)
2. **`ResizeObserver` fires** — `pages-component-terminal` observes its container via `ResizeObserver`
3. **xterm.js `fit()`** — recalculates terminal cols/rows from container dimensions
4. **`terminal-resize` event** — dispatched by `pages-component-terminal` with new cols/rows
5. **`TerminalHandle.resize(cols, rows)`** — calls `POST /api/sessions/{id}/resize?cols=&rows=`
6. **`TmuxService.resizeWindow()`** — delivers SIGWINCH to the tmux session so TUI apps redraw

Critical transitions requiring resize:
- **Tablet:** Nav collapse (220px → 48px icon rail) changes terminal width
- **Phone tab switch:** Terminal gains full viewport width when becoming the active tab
- **Orientation change:** Portrait resize events fire even without landscape support

CSS-driven layout changes that alter the terminal container's dimensions automatically trigger the `ResizeObserver` → `fit()` → `terminal-resize` chain. For phone tab switches, `visibility: hidden` panels maintain their dimensions at full size, so switching to the terminal tab should not trigger a dimension change. The `_switchTab()` handler calls `fit()` defensively via `requestAnimationFrame` to cover edge cases (e.g., orientation change while on a different tab).

## Outer Shell Adaptations (session.html)

### Tablet
- Header padding: 16px -> 12px
- Toggle buttons: icon-only (remove labels)
- Session name: allow truncation with ellipsis

### Phone
- Header becomes single-row compact: hamburger + session name (truncated) + status dot
- Back link: icon only (no "Dashboard" text)
- Toggle buttons: move to bottom tab bar or hamburger menu
- Touch targets: minimum 44px height

## Fleet Home Adaptations (index.html)

### pages-runtime Layout Interaction

The fleet home page uses `columns(hostPanel("fleet-panel"), hostPanel("session-grid"), hostPanel("mesh-panel"))` from pages-runtime. Each panel is a separate custom element with its own Shadow DOM CSS. The `columns()` primitive produces a flex/grid container at the `#app` level.

Individual component media queries on `:host` (e.g., `claudony-fleet-panel`'s `display: none` at phone breakpoint) work correctly alongside the `columns()` layout — the existing 600px breakpoints (being migrated to 768px) demonstrate this pattern is already proven. The grid container accommodates hidden/collapsed children via flex layout.

Similarly, the session page uses `rows(hostPanel("terminal-header"), hostPanel("terminal-workspace"), hostPanel("key-bar"))`. The `rows()` layout is fully compatible with responsive media queries on individual panels.

### Tablet
- Session grid: `minmax(240px, 1fr)` (from 280px). Cards slightly narrower.
- Fleet panel and mesh panel: stack vertically if sidebar active.

### Phone
- Session grid: single column. Cards full-width.
- Header: hamburger menu for fleet/mesh panels.
- "New Session" button: floating action button (bottom-right) or in header.

## Components Affected

| Component | Changes |
|-----------|---------|
| `claudony-workbench.ts` | Shadow DOM media queries for 3-tier panel layout, icon rail, `_activeTab` state, inline tab bar, `_navDrawerOpen` state for tablet overlay |
| `terminal-workspace.ts` | Shadow DOM media queries for tablet/phone, `_activeTab` state, inline tab bar, `.tab-panel` wrapper in phone render |
| `worker-panel.ts` | `:host` media query for tablet icon rail (48px) and overlay positioning (`position: fixed`) |
| `channel-panel.ts` | `:host` media query for tablet overlay drawer (`position: fixed; right: 0`) |
| `session-grid.ts` | Grid column adjustments for tablet/phone |
| `terminal-header.ts` | Compact header, icon-only buttons, hamburger |
| `claudony-fleet-panel.ts` | Migrate `@media (max-width: 600px)` breakpoint to `768px` |
| `claudony-mesh-panel.ts` | Stack/collapse behaviour |
| `style.css` | Migrate 600px breakpoints to 768px, outer shell header, grid, FAB |
| `key-bar.ts` | Touch target sizing via media query, hide on non-terminal phone tabs via `active-tab-changed` event |

**Note:** Both `session.html` and `index.html` already have `<meta name="viewport" content="width=device-width, initial-scale=1">`. Update both to add `viewport-fit=cover`:
```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```
This is required for `env(safe-area-inset-bottom)` to return non-zero values on notched iPhones, enabling the tab bar to clear the home indicator.

## Touch Considerations

- All interactive elements: minimum 44x44px touch target on tablet/phone.
- **`key-bar.ts`:** Already auto-shows on touch devices (`'ontouchstart' in window`). Buttons currently use `pages-button size="xs"` which is below 44px. Add Shadow DOM media query:
  ```css
  @media (max-width: 1024px) {
    pages-button { min-height: 44px; min-width: 44px; }
    .key-bar { padding: 4px 8px; gap: 4px; }
  }
  ```
- Channel feed: scroll momentum, pull-to-refresh for new messages (later).
- Terminal: pinch-to-zoom (xterm.js supports this natively).

## Testing Strategy

- **CSS changes:** Visual testing via Playwright at 3 viewport sizes (1280x800, 768x1024, 375x812).
- **JS state changes:** Unit tests for `_activeTab` switching, `_switchTab()` handler, tab bar rendering per breakpoint.
- **Terminal resize:** Integration test verifying `terminal-resize` event fires after layout transition.
- **E2E tests:** Existing tests run at desktop resolution. Add viewport-parameterised variants for critical paths (workbench renders, tab switching works, overlay drawers open).

## Out of Scope

- Swipe gestures between panels — tracked as claudony#194.
- Offline/PWA enhancements — tracked as claudony#195.
- Landscape phone optimisations (portrait sufficient for now) — tracked as claudony#196.
- Responsive typography scaling — tracked as claudony#197.
