# Responsive Typography Scaling + Landscape Phone Optimisations

**Issues:** #197, #196
**Date:** 2026-08-04
**Status:** Approved

## Context

Claudony's frontend has two responsive gaps deferred from the responsive layouts epic (#179):

1. **Typography** — all font sizes are hardcoded as px or rem values across 12+ files. The `pages-ui-tokens` design system provides `--pages-font-size-xs`, `--pages-font-size-sm`, `--pages-font-size-base`, `--pages-font-size-lg` but Claudony's own components don't use them. Only embedded blocks-ui components use tokens.

2. **Landscape phone** — at the 767px breakpoint, the terminal page uses stacked tab panels (terminal vs chat). In landscape phone orientation (~375px height), the terminal gets minimal vertical space after header (44px), tab bar (48px + safe-area-inset), and browser chrome eat the viewport.

## Design

### Part 1: Typography Token Migration (#197)

**Approach:** Replace all hardcoded font sizes in Claudony components with `--pages-font-size-*` tokens, then add breakpoint overrides so tokens scale across viewports.

#### Token Mapping

| Current value | Token | Used for |
|---|---|---|
| 9px, 10px | `--pages-font-size-xs` (10px) | Chevrons, peer source badges, circuit labels, dock controls |
| 11px, 12px | `--pages-font-size-sm` (12px) | Labels, metadata, timestamps, panel headers, badges |
| 13px, 14px | `--pages-font-size-base` (14px) | Body text, card names, buttons, input fields, worker rows |
| 15px, 16px, 18px | `--pages-font-size-lg` (18px) | Page headings (h1, h2), back arrow |

#### Breakpoint Overrides

Added in two places:
- `THEME_CSS` in `theme.ts` — scoped to `:host`, inherited by all shadow-DOM components
- `style.css` — scoped to `:root`, covers the global stylesheet (fleet home, terminal page shell)

Shadow DOM components get overrides via `THEME_CSS`; light-DOM elements in `index.html`/`session.html` get overrides via `style.css`.

```css
@media (max-width: 767px) {
  :host {
    --pages-font-size-xs: 9px;
    --pages-font-size-sm: 11px;
    --pages-font-size-base: 13px;
    --pages-font-size-lg: 16px;
  }
}
@media (min-width: 1440px) {
  :host {
    --pages-font-size-xs: 11px;
    --pages-font-size-sm: 13px;
    --pages-font-size-base: 15px;
    --pages-font-size-lg: 20px;
  }
}
```

#### Files Affected

**Token replacement (hardcoded → token):**
- `theme.ts` — add breakpoint overrides to `THEME_CSS`
- `session-panel.ts` — h2 (1.1rem→lg), card-name (0.95rem→base), card-dir (0.8rem→sm), card-time (0.75rem→xs), auth-body p (0.9rem→base), view-toggle button (14px→base)
- `terminal-header.ts` — session-name (14px→base, 13px→sm at 1024px, 12px→sm at 767px), back arrow (16px→lg)
- `terminal-workspace.ts` — tab-btn (12px→sm)
- `worker-panel.ts` — header (12px→sm), worker-item (13px→base), worker-time (11px→sm), empty (13px→base)
- `channel-panel.ts` — channel-select (12px→sm), tab buttons (13px→base)
- `claudony-fleet-panel.ts` — header (12px→sm), peer-empty (12px→sm), peer-card (12px→sm), peer-name (12px→sm), peer-source (10px→xs), peer-actions button (10px→xs)
- `claudony-workbench.ts` — dock-btn (11px→sm), case-role (12px→sm), case-elapsed (11px→sm), lineage-toggle (11px→sm), chevron (9px→xs), lineage-row (11px→sm), stale-prompt (12px→sm), stale-btn (11px→sm), error (11px→sm), worker-row (13px→base)
- `claudony-mesh-panel.ts` — all mesh labels, channel names, feed items, dock controls
- `claudony-action-inbox.ts` — summary (0.875rem→base), table (0.8125rem→sm), actions button (0.75rem→xs)
- `claudony-case-browser.ts` — if any hardcoded sizes exist
- `key-bar.ts` — key labels
- `style.css` — global stylesheet: h1 (18px→lg), button (14px→base), card-name (15px→lg), badge (11px→sm), card-dir (12px→sm), card-meta (11px→sm), card-actions button (12px→sm), dialog (16px→lg, 13px→sm, 14px→base), compose (14px→base, 11px→sm), mesh sizes, dock sizes

**No changes needed:**
- Embedded blocks-ui components (`channel-feed`, `channel-input`, `case-explorer`) — already use tokens internally; inherit overrides automatically.

### Part 2: Landscape Phone Optimisation (#196)

**Approach:** Hide header and tab bar chrome in phone landscape to maximise terminal height. Provide a floating overlay for navigation.

#### Trigger

```css
@media (orientation: landscape) and (max-height: 500px)
```

Catches phones in landscape (~375px height) but not tablets (iPad Mini landscape = 744px).

#### Changes — Terminal Page Only

1. **Hide `terminal-header`** — the back link, session name, and status badge disappear. Saves ~44px.

2. **Hide the bottom tab bar** in `terminal-workspace` — the Terminal/Chat toggle disappears. Saves ~48px + safe-area-inset.

3. **Floating nav overlay** — a small fixed-position cluster (top-left, semi-transparent) with:
   - **Back button** — 32px circle, navigates to fleet home
   - **Chat toggle** — 32px circle, only visible when session has a caseId. Toggles between terminal and chat full-screen.
   - Fades in on touch of the top-left zone, auto-fades after 3s of inactivity.

4. **Implementation location** — inline in `terminal-workspace.ts` (the overlay is 2 buttons and a fade timer, not worth a separate component).

#### What Doesn't Change

- Fleet home (`index.html`) — cards reflow naturally in landscape.
- Desktop and tablet layouts — the `max-height: 500px` guard excludes them.
- Workbench layout — case-bound sessions with the three-column workbench are desktop/tablet only.

## Testing

- **Typography:** Visual inspection at 3 breakpoints (phone 375px, desktop 1440px, default). Verify blocks-ui components inherit token overrides. Existing vitest unit tests should pass without changes (they don't test font sizes).
- **Landscape:** Playwright test with `page.setViewportSize({width: 667, height: 375})` to simulate phone landscape. Verify header hidden, floating nav visible, terminal fills viewport. Test chat toggle if caseId present.
- **Regression:** Run full vitest suite + existing Playwright E2E suite to verify no breakage.
