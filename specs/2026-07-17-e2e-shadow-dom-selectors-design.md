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

**Shadow DOM piercing rule:** Playwright `page.locator()` CSS selectors pierce
open shadow DOM automatically. `page.evaluate()` JavaScript does not — any
`evaluate()` call that needs to reach elements inside shadow DOM must explicitly
traverse `element.shadowRoot.querySelector(...)`. Existing ChannelPanelE2ETest
evaluate calls already use this pattern (e.g. `panel.shadowRoot.querySelector('.ch-select')`).

**Scope note:** Issue #172 acceptance states "no test logic changes — only
selector updates." Two deviations are mechanically forced by the DOM change
and cannot be avoided:
1. A production fix in `connectedCallback()` is a prerequisite — without it,
   two toggle tests fail because the host element lacks the `collapsed` class
   on initial load (see §Production Fix).
2. Lit's conditional rendering (`nothing`) replaces CSS class toggling for
   lineage visibility, and the lineage count element was removed in favour of
   inline text. These structural assertion changes are mechanically forced —
   the old DOM elements no longer exist (see §CaseContextPanelE2ETest).

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

This is the only production code change. It is required for the E2E tests:
`toggleBtn_opensThenClosesPanel` and `ctrlK_togglesPanelOpenAndClosed` assert
`getAttribute("class").contains("collapsed")` on initial page load. Without
this fix, `_collapsed = true` but no `collapsed` class exists on the host
element until `close()` is explicitly called.

## CaseContextPanelE2ETest (4 tests)

### Selector renames (mechanical)

| Old | New | Occurrences | Reason |
|-----|-----|-------------|--------|
| `.ch-case-header` | `.case-header` | 3 (lines 76, 77, 98) | `ch-` prefix dropped in shadow DOM |
| `.ch-case-role` | `.case-role` | 1 (line 80) | same |
| `.ch-case-elapsed` | `.case-elapsed` | 1 (line 87) | same |
| `.ch-case-header .worker-status-dot` | `.case-header .status-dot` | 1 (line 83) | both renamed |
| `.ch-lineage-toggle` | `.lineage-toggle` | 3 (lines 107, 113, 120) | same |

### Structural changes

1. **Lineage visibility** — old: `.ch-lineage` class toggled `.ch-lineage-hidden`.
   New: Lit conditional rendering via `nothing` — the `.lineage` element is not
   rendered when collapsed (`${this._lineageExpanded ? this._renderLineage() : nothing}`).
   Assert `.lineage` element count (0 = collapsed, 1 = expanded) instead of
   checking a hidden class. The old DOM structure no longer exists — this change
   is mechanically forced. (3 occurrences: lines 110, 114, 121)

2. **Lineage count** — old: `.ch-lineage-count` dedicated element. New: inline
   text in second `<span>` inside `.lineage-toggle` (the toggle renders
   `` `${count} prior worker${...}` `` directly). Assert `.lineage-toggle`
   textContent contains "0 prior workers". The dedicated element was removed —
   this change is mechanically forced. (1 occurrence: line 117)

3. **Channel select locator** — old: `#ch-select` (id selector). New: `.ch-select`
   (class selector) inside shadow DOM. The `<select>` element changed from
   `id="ch-select"` to `class="ch-select"` in the Lit template.
   `page.locator("#ch-select option[value='...']")` → `page.locator(".ch-select option[value='...']")`.
   (1 occurrence: line 147)

4. **Channel select evaluate** — old: `document.getElementById('ch-select')`.
   New: `document.querySelector('claudony-channel-panel').shadowRoot.querySelector('.ch-select')`.
   `getElementById()` doesn't pierce shadow DOM. (1 occurrence: line 150)

**Total: 9 selector renames + 6 structural changes = 15 individual updates.**

### Helper method

`openChannelPanel()` uses `#channel-panel:not(.collapsed)` — works as-is
(Playwright matches the custom element host in the light DOM).

## ChannelPanelE2ETest (19 tests)

### Full test disposition

| # | Test | Disposition | Detail |
|---|------|-------------|--------|
| 1 | `toggleBtn_opensThenClosesPanel` | Production fix | Asserts `collapsed` class on load |
| 2 | `ctrlK_togglesPanelOpenAndClosed` | Production fix | Same assertion pattern |
| 3 | `messageBadges_showCorrectTypeLabel` | Verify badge class | See §Badge class note |
| 4 | `channelDropdown_populatesWithAvailableChannels` | No change | Uses `panelShadow()` helper |
| 5 | `channelPreselect_loadsTimelineMessages` | No change | Uses `feedMessages()`/`feedText()` |
| 6 | `humanSender_rendersWithHumanActorIcon` | No change | Uses `feedText()` |
| 7 | `interjectionDock_postMessageAppearsInFeed` | No change | Uses helpers + shadow root evaluate |
| 8 | `cursorPolling_onlyNewMessagesAppearAfterInitialLoad` | No change | Uses `feedMessages()`/`feedText()` |
| 9 | `eventMessage_rendersWithEventBadgeAndTelemetryFields` | No change | Uses `feedMessages()`/`feedText()` |
| 10 | `eventMessage_withMissingTelemetryFields_rendersDash` | No change | Uses `feedMessages()` |
| 11 | `interjectionDock_defaultTypeIsCommand` | No change | Uses `panelShadow()` |
| 12 | `interjectionDock_typeDropdown_filteredToChannelAllowedTypes` | No change | Uses helpers + shadow root evaluate |
| 13 | `catchUp_onPanelReopen_usesAfterCursor` | No change | Uses `feedMessages()`/`feedText()` |
| 14 | `staleCursor_showsReconnectPrompt` | No change | Uses `panelShadow()` |
| 15 | `staleCursor_chooseCatchUp_fetchesFromCursor` | No change | Uses `panelShadow()`/`feedMessages()` |
| 16 | `staleCursor_chooseReload_fetchesFullHistory` | No change | Uses `panelShadow()`/`feedMessages()`/`feedText()` |
| 17 | `interjectionDock_openChannel_showsAllTypes` | No change | Uses helpers + shadow root evaluate |
| 18 | `channelEvents_pushesMessageInRealTime` | No change | Uses `panelShadow()`/`feedMessages()`/`feedText()` |
| 19 | `channelPanel_eventSourceError_fallsBackToPoll` | No change | Uses `panelShadow()`/`feedMessages()`/`feedText()` |

**16 no change, 2 production fix, 1 verify badge class.**

### Badge class note

`messageBadges_showCorrectTypeLabel` uses the selector chain
`channel-feed >> channel-message >> .msg-badge`. The `channel-message`
component is from `@casehubio/blocks-ui-channel-activity` (v0.2.2). The
package source is not available in any indexed repository for verification.
The badge class name must be verified at implementation time via browser
DevTools inspection of the rendered `channel-message` component. If `.msg-badge`
is still the correct class in the current package version, no test change is
needed for this selector.

## TerminalPageE2ETest (2 tests)

No changes needed. Both tests use DOM elements outside channel-panel's
shadow DOM:

- `terminalPage_loadsStructure...` queries `pages-component-terminal`,
  `#status-badge`, `#session-name` — all light DOM selectors on the terminal
  page layout, unrelated to channel-panel.
- `proxyMode_resizeCallsProxyEndpoint` uses route interception and
  `window._xtermTerminal.resize()` — no channel-panel DOM involvement.
  `terminal-workspace.ts` uses light DOM (`innerHTML`), and the resize test
  intercepts fetch URLs with no shadow DOM selectors.

Issue #172 listed "Update proxy-resize test in TerminalPageE2ETest" based on
initial triage. Investigation confirms neither test touches shadow DOM. Issue
\#172 should be updated to remove this line item.

## Change Summary

| File | Changes |
|------|---------|
| `channel-panel.ts` | 1 line — initial `collapsed` class in `connectedCallback()` |
| `CaseContextPanelE2ETest.java` | 9 selector renames + 6 structural changes (15 total) |
| `ChannelPanelE2ETest.java` | 0–1 selector rename (pending badge class verification) |
| `TerminalPageE2ETest.java` | 0 |

## Verification

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e \
  -Dtest=ChannelPanelE2ETest,CaseContextPanelE2ETest,TerminalPageE2ETest
```

**Prerequisite:** verify the `.msg-badge` class in `channel-message` via browser
DevTools before running (see §Badge class note). If the class has changed, update
the selector in `messageBadges_showCorrectTypeLabel` first.

With the correct badge selector in place, all E2E tests pass. Unit tests
(`mvn test`) still pass (production change is additive — no logic change).
