# E2E Shadow DOM Selector Migration — Design Spec

**Issue:** casehubio/claudony#172
**Date:** 2026-07-17
**Status:** Approved

## Context

\#170 refactored `channel-panel.ts` from a vanilla HTMLElement to a LitElement
with Shadow DOM, and adopted blocks-ui `<channel-feed>` and `<channel-input>`
components. The E2E tests that query channel-panel, case-context, and related
DOM structure now fail because selectors target the old light DOM structure.

## Approach

Approach A: fix the production bug + update selectors inline. No page objects,
no data-testid attributes — Playwright pierces open shadow DOM by default, so
the existing locator patterns work once selectors match the new class names.

## Production Fix

**channel-panel.ts** — `connectedCallback()` must set the initial `collapsed`
class on the host element to match `_collapsed = true`:

```typescript
connectedCallback(): void {
    super.connectedCallback();
    this.classList.add('collapsed');
    this._loadCursors();
    this._fetchMeshConfig();
}
```

This is the only production code change.

## CaseContextPanelE2ETest (4 tests)

### Selector renames (mechanical)

| Old | New | Reason |
|-----|-----|--------|
| `.ch-case-header` | `.case-header` | `ch-` prefix dropped in shadow DOM |
| `.ch-case-role` | `.case-role` | same |
| `.ch-case-elapsed` | `.case-elapsed` | same |
| `.ch-case-header .worker-status-dot` | `.case-header .status-dot` | both renamed |
| `.ch-lineage-toggle` | `.lineage-toggle` | same |

### Structural changes

1. **Lineage visibility** — old: `.ch-lineage` class toggled `.ch-lineage-hidden`.
   New: conditional rendering via Lit `nothing`. Assert `.lineage` element
   count (0 = collapsed, 1 = expanded) instead of checking a hidden class.

2. **Lineage count** — old: `.ch-lineage-count` dedicated element. New: inline
   text in second `<span>` inside `.lineage-toggle`. Assert `.lineage-toggle`
   textContent contains "0 prior workers".

3. **Channel select** — old: `#ch-select` (id). New: `.ch-select` (class) inside
   shadow DOM. `document.getElementById()` doesn't pierce shadow DOM; replace
   with `document.querySelector('claudony-channel-panel').shadowRoot.querySelector('.ch-select')`.

### Helper method

`openChannelPanel()` uses `#channel-panel:not(.collapsed)` — works as-is
(Playwright matches the custom element host).

## ChannelPanelE2ETest (19 tests)

### Already correct (16 tests)

Helper methods `panelShadow()`, `feedMessages()`, `feedText()` use correct
selectors. Most tests use these helpers or selectors that already match the
new DOM.

### Changed by production fix (2 tests)

`toggleBtn_opensThenClosesPanel` and `ctrlK_togglesPanelOpenAndClosed` assert
`#channel-panel` class contains "collapsed" on initial load. The production fix
(adding `collapsed` in `connectedCallback`) makes these pass without test changes.

### Selector change (1 test)

`messageBadges_showCorrectTypeLabel`: `.msg-badge` → `.speech-act-badge`
(blocks-ui `channel-message` uses `speech-act-badge` class).

## TerminalPageE2ETest (2 tests)

No changes needed. `terminalPage_loadsStructure` uses terminal-header light DOM
selectors. `proxyMode_resizeCallsProxyEndpoint` uses route interception with no
shadow DOM involvement. If proxy-resize fails, it's a different issue (not #172).

## Change Summary

| File | Changes |
|------|---------|
| `channel-panel.ts` | 1 line — initial `collapsed` class |
| `CaseContextPanelE2ETest.java` | ~15 selector updates + 2 structural adjustments |
| `ChannelPanelE2ETest.java` | 1 selector rename (`.msg-badge` → `.speech-act-badge`) |
| `TerminalPageE2ETest.java` | 0 |

## Verification

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e \
  -Dtest=ChannelPanelE2ETest,CaseContextPanelE2ETest,TerminalPageE2ETest
```

All E2E tests green. Unit tests (`mvn test`) still pass (production change is
additive — no logic change).
