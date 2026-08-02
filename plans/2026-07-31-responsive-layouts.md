# Responsive Layouts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #179 — Responsive layouts for tablet/phone
**Issue group:** #188, #179

**Goal:** Add progressive responsive degradation to all Claudony UI components across three tiers: desktop (>1024px), tablet (768-1024px), phone (<768px).

**Architecture:** CSS media queries in Shadow DOM for component-level layout, plus media queries in `style.css` for the outer shell. Minimal JS state (`_activeTab`) for phone tab switching only. Two workbench components (`claudony-workbench` for case-bound, `terminal-workspace` for standalone) adapt independently. Terminal preservation via `visibility: hidden` on inactive tab panels.

**Tech Stack:** Lit 3 (Shadow DOM CSS + reactive state), CSS media queries, Playwright (viewport-parameterised E2E tests)

## Global Constraints

- Breakpoints: phone <768px, tablet 768-1024px, desktop >1024px
- Touch targets: minimum 44x44px on tablet/phone
- Terminal must not be destroyed/recreated on tab switch — use `visibility: hidden`
- No `matchMedia()` or `ResizeObserver` for tier detection
- No swipe gestures
- All edits to `.ts`/`.tsx` files via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_replace_text_in_file`)

---

### Task 1: Foundation — breakpoint migration + viewport meta

**Files:**
- Modify: `app/src/main/resources/META-INF/resources/app/style.css` (lines 168, 363)
- Modify: `app/src/main/webui/src/components/claudony-fleet-panel.ts` (line 79)
- Modify: `app/src/main/resources/META-INF/resources/app/session.html`
- Modify: `app/src/main/resources/META-INF/resources/app/index.html`

**Interfaces:**
- Consumes: nothing
- Produces: consistent 768px phone breakpoint across all existing responsive rules; `viewport-fit=cover` enabling `env(safe-area-inset-bottom)`

- [ ] **Step 1: Migrate style.css breakpoint at line 168**

```css
/* Before: @media (max-width: 600px) { */
/* After:  @media (max-width: 767px) { */
```

Use `ide_replace_text_in_file` on `app/src/main/resources/META-INF/resources/app/style.css`:
- Search: `@media (max-width: 600px) {\n    #session-grid`
- Replace: `@media (max-width: 767px) {\n    #session-grid`

- [ ] **Step 2: Migrate style.css breakpoint at line 363**

```css
/* Before: @media (max-width: 600px) { */
/* After:  @media (max-width: 767px) { */
```

Use `ide_replace_text_in_file` — search for the fleet responsive block:
- Search: `@media (max-width: 600px) {\n    #fleet-panel`
- Replace: `@media (max-width: 767px) {\n    #fleet-panel`

- [ ] **Step 3: Migrate fleet-panel Shadow DOM breakpoint**

Use `ide_replace_text_in_file` on `claudony-fleet-panel.ts`:
- Search: `@media (max-width: 600px) { :host { display: none; } }`
- Replace: `@media (max-width: 767px) { :host { display: none; } }`

- [ ] **Step 4: Update viewport meta in both HTML files**

Use `ide_replace_text_in_file` on `session.html`:
- Search: `content="width=device-width, initial-scale=1"`
- Replace: `content="width=device-width, initial-scale=1, viewport-fit=cover"`

Repeat for `index.html`.

- [ ] **Step 5: Verify no remaining 600px breakpoints**

```bash
grep -rn "600px" app/src/main/resources/META-INF/resources/app/style.css app/src/main/webui/src/
```

Expected: zero matches.

- [ ] **Step 6: Commit**

```
chore(#179): migrate breakpoints 600px→768px and add viewport-fit=cover
```

---

### Task 2: Fleet home responsive — session grid + outer shell

**Files:**
- Modify: `app/src/main/webui/src/components/session-grid.ts` (Shadow DOM styles)
- Modify: `app/src/main/resources/META-INF/resources/app/style.css`

**Interfaces:**
- Consumes: 768px breakpoint from Task 1
- Produces: tablet grid columns at 240px min-width; phone single-column grid

- [ ] **Step 1: Add tablet breakpoint to session-grid Shadow DOM**

Read `session-grid.ts` styles section. Add media query after existing grid styles:

```css
@media (max-width: 1024px) {
  .session-grid { grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 12px; padding: 16px; }
}
@media (max-width: 767px) {
  .session-grid { grid-template-columns: 1fr; gap: 12px; padding: 12px; }
}
```

Use `ide_replace_text_in_file` to append inside the `static styles = css\`...\`` block.

- [ ] **Step 2: Add tablet header padding to style.css**

Add between the existing phone rule and the fleet layout section:

```css
@media (max-width: 1024px) {
    header { padding: 12px 16px; }
}
```

- [ ] **Step 3: Visual check at 3 viewports**

Open the fleet home in a browser at 1280x800, 768x1024, 375x812 and verify:
- Desktop: unchanged layout
- Tablet: slightly narrower cards, compact header padding
- Phone: single-column cards, fleet panel hidden

- [ ] **Step 4: Commit**

```
feat(#179): fleet home responsive — tablet/phone grid and header
```

---

### Task 3: Terminal header responsive

**Files:**
- Modify: `app/src/main/webui/src/components/terminal-header.ts`

**Interfaces:**
- Consumes: 768px/1024px breakpoints from Task 1
- Produces: compact header at tablet (icon-only buttons), minimal header at phone (truncated name, icon back link)

- [ ] **Step 1: Add tablet and phone media queries to Shadow DOM styles**

Use `ide_replace_text_in_file` on `terminal-header.ts`. Append before the closing backtick of `static override styles = css\`...\``:

```css
@media (max-width: 1024px) {
  .terminal-header { padding: 8px 12px; gap: 8px; }
  .back-link { font-size: 0; }
  .back-link::before { content: '\\2190'; font-size: 14px; }
  .session-name { font-size: 13px; }
}
@media (max-width: 767px) {
  .terminal-header { padding: 6px 10px; gap: 6px; }
  .session-name { font-size: 12px; max-width: 40vw; }
}
```

- [ ] **Step 2: Ensure toggle buttons have minimum touch targets on tablet**

Add to the tablet media query:

```css
@media (max-width: 1024px) {
  pages-button { min-height: 44px; }
}
```

- [ ] **Step 3: Commit**

```
feat(#179): terminal header responsive — compact tablet/phone
```

---

### Task 4: Workbench tablet layout

**Files:**
- Modify: `app/src/main/webui/src/components/claudony-workbench.ts` (Shadow DOM styles + `_navDrawerOpen` state + render method)

**Interfaces:**
- Consumes: 1024px tablet breakpoint
- Produces: nav panel collapses to 48px icon rail; conversation area narrows to 320px; context panel hidden (case header moves to conversation area); nav drawer overlay on tap

- [ ] **Step 1: Add `_navDrawerOpen` reactive state**

Use `ide_insert_member` in `ClaudonyWorkbench` after `_dockState`:

```typescript
@state() private _navDrawerOpen = false;
```

- [ ] **Step 2: Add tablet media queries to Shadow DOM styles**

Use `ide_replace_text_in_file` to append tablet rules before the closing backtick of styles. The nav panel collapses, conversation narrows, context panel hides:

```css
@media (max-width: 1024px) {
  .nav-panel { width: 48px; min-width: 48px; overflow: visible; }
  .nav-panel .worker-list { display: none; }
  .nav-panel .nav-icon { display: flex; }
  .nav-drawer-overlay {
    position: absolute; left: 48px; top: 0; bottom: 0;
    width: 220px; z-index: 10;
    background: var(--pages-neutral-2, #252526);
    border-right: 1px solid var(--pages-neutral-4, #3e3e42);
    box-shadow: 4px 0 8px rgba(0,0,0,0.3);
  }
  .conversation-area { width: 320px; min-width: 320px; }
  .context-panel { display: none; }
}
```

- [ ] **Step 3: Add nav icon and drawer overlay to render method**

Modify `_renderWorkerNav()` to include an icon rail element and conditional drawer overlay. The icon toggles `_navDrawerOpen`. When open, the drawer overlays the terminal area.

- [ ] **Step 4: Visual check — tablet viewport**

Open a case-bound session at 768x1024:
- Nav panel shows icon rail (48px)
- Tapping icon opens overlay drawer with worker list
- Conversation area is 320px
- Context panel is hidden
- Terminal resizes correctly

- [ ] **Step 5: Commit**

```
feat(#179): workbench tablet — nav icon rail + narrow conversation
```

---

### Task 5: Workbench phone layout

**Files:**
- Modify: `app/src/main/webui/src/components/claudony-workbench.ts` (Shadow DOM styles + `_activeTab` state + render method + `_switchTab` handler)

**Interfaces:**
- Consumes: tablet layout from Task 4
- Produces: full-screen tab panels (chat/terminal/context) with bottom tab bar; `_activeTab` state; `active-tab-changed` pages-event

- [ ] **Step 1: Add `_activeTab` reactive state**

Use `ide_insert_member` after `_navDrawerOpen`:

```typescript
@state() private _activeTab: 'chat' | 'terminal' | 'context' = 'chat';
```

- [ ] **Step 2: Add `_switchTab` handler**

Use `ide_insert_member`:

```typescript
private _switchTab(tab: 'chat' | 'terminal' | 'context'): void {
  this._activeTab = tab;
  this.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true, composed: true,
    detail: { topic: 'active-tab-changed', payload: { tab } },
  }));
  if (tab === 'terminal') {
    requestAnimationFrame(() => {
      const term = this.renderRoot.querySelector('pages-component-terminal');
      if (term && 'fit' in term) (term as { fit(): void }).fit();
    });
  }
}
```

- [ ] **Step 3: Add phone media queries to Shadow DOM styles**

```css
@media (max-width: 767px) {
  :host { flex-direction: column; }
  .nav-panel { display: none; }
  .conversation-area { width: 100%; min-width: unset; border-left: none; }
  .context-panel { display: none; }
  .main-panel { position: absolute; inset: 0; visibility: hidden; }
  .main-panel.active { visibility: visible; z-index: 1; }
  .conversation-area { position: absolute; inset: 0; }
  .conversation-area.active { z-index: 1; }

  .tab-content { position: relative; flex: 1; overflow: hidden; }
  .tab-panel { position: absolute; inset: 0; visibility: hidden; }
  .tab-panel.active { visibility: visible; z-index: 1; }

  .tab-bar {
    display: flex; height: 48px;
    border-top: 1px solid var(--pages-neutral-4, #3e3e42);
    background: var(--pages-neutral-2, #252526);
    padding-bottom: env(safe-area-inset-bottom);
    flex-shrink: 0;
  }
  .tab-btn {
    flex: 1; display: flex; align-items: center; justify-content: center;
    background: none; border: none; color: var(--pages-neutral-8, #888);
    font-size: 12px; cursor: pointer; min-height: 44px;
  }
  .tab-btn[aria-selected="true"] { color: var(--pages-accent-9, #6366f1); }
}
@media (min-width: 768px) {
  .tab-bar { display: none; }
}
```

- [ ] **Step 4: Restructure render method for phone tab panels**

Wrap the terminal and conversation areas in `.tab-panel` divs with conditional `.active` class based on `_activeTab`. Add the `.tab-bar` nav element at the bottom of the render. The tab bar is always rendered but hidden via CSS on desktop/tablet.

- [ ] **Step 5: Visual check — phone viewport**

Open a case-bound session at 375x812:
- Chat tab is default — channel feed + input fill the screen
- Terminal tab shows full-screen terminal
- Context tab shows case header + lineage
- Tab bar at bottom with 3 tabs
- Switching tabs preserves terminal state (no reconnect)

- [ ] **Step 6: Commit**

```
feat(#179): workbench phone — tab bar + full-screen panels
```

---

### Task 6: Terminal-workspace tablet layout

**Files:**
- Modify: `app/src/main/webui/src/components/terminal-workspace.ts` (Shadow DOM styles)
- Modify: `app/src/main/webui/src/components/worker-panel.ts` (`:host` media query for icon rail + overlay)
- Modify: `app/src/main/webui/src/components/channel-panel.ts` (`:host` media query for overlay drawer)

**Interfaces:**
- Consumes: 1024px tablet breakpoint
- Produces: worker panel as 48px icon rail with fixed overlay; channel panel as fixed overlay drawer; terminal gets remaining space

- [ ] **Step 1: Add tablet styles to worker-panel Shadow DOM**

Read `worker-panel.ts` styles. Append `:host` media query for tablet — collapses to icon rail, expands to fixed overlay:

```css
@media (max-width: 1024px) {
  :host { width: 48px; min-width: 48px; overflow: visible; }
  :host(.expanded) {
    position: fixed; z-index: 100; left: 0; top: 0; bottom: 0;
    width: 240px; box-shadow: 4px 0 8px rgba(0,0,0,0.3);
  }
}
```

- [ ] **Step 2: Add tablet styles to channel-panel Shadow DOM**

Read `channel-panel.ts` styles. Append `:host` media query:

```css
@media (max-width: 1024px) {
  :host { display: none; }
  :host(.expanded) {
    display: flex; position: fixed; z-index: 100;
    right: 0; top: 0; bottom: 0; width: 300px;
    box-shadow: -4px 0 8px rgba(0,0,0,0.3);
  }
}
```

- [ ] **Step 3: Add tablet styles to terminal-workspace Shadow DOM**

```css
@media (max-width: 1024px) {
  #terminal-container { flex: 1; }
}
```

- [ ] **Step 4: Visual check — standalone session at tablet viewport**

Open a standalone session at 768x1024:
- Terminal fills the screen
- Worker panel collapsed to icon rail
- Channel panel hidden, accessible via header toggle

- [ ] **Step 5: Commit**

```
feat(#179): terminal-workspace tablet — icon rail + overlay panels
```

---

### Task 7: Terminal-workspace phone layout

**Files:**
- Modify: `app/src/main/webui/src/components/terminal-workspace.ts` (styles + `_activeTab` state + render restructure + `_switchTab` handler)

**Interfaces:**
- Consumes: tablet layout from Task 6
- Produces: full-screen tab panels (terminal/chat) with bottom tab bar; `_activeTab` state; `active-tab-changed` pages-event

- [ ] **Step 1: Add `_activeTab` state and `_switchTab` handler**

Use `ide_insert_member`:

```typescript
@state() private _activeTab: 'terminal' | 'chat' = 'terminal';

private _switchTab(tab: 'terminal' | 'chat'): void {
  this._activeTab = tab;
  this.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true, composed: true,
    detail: { topic: 'active-tab-changed', payload: { tab } },
  }));
  if (tab === 'terminal') {
    requestAnimationFrame(() => {
      const term = this.renderRoot.querySelector('pages-component-terminal');
      if (term && 'fit' in term) (term as { fit(): void }).fit();
    });
  }
}
```

- [ ] **Step 2: Add phone styles to Shadow DOM**

```css
@media (max-width: 767px) {
  :host { flex-direction: column; }
  .tab-content { position: relative; flex: 1; overflow: hidden; }
  .tab-panel { position: absolute; inset: 0; visibility: hidden; }
  .tab-panel.active { visibility: visible; z-index: 1; }
  .tab-bar {
    display: flex; height: 48px;
    border-top: 1px solid var(--pages-neutral-4, #3e3e42);
    background: var(--pages-neutral-2, #252526);
    padding-bottom: env(safe-area-inset-bottom);
    flex-shrink: 0;
  }
  .tab-btn {
    flex: 1; display: flex; align-items: center; justify-content: center;
    background: none; border: none; color: var(--pages-neutral-8, #888);
    font-size: 12px; cursor: pointer; min-height: 44px;
  }
  .tab-btn[aria-selected="true"] { color: var(--pages-accent-9, #6366f1); }
}
@media (min-width: 768px) {
  .tab-bar { display: none; }
}
```

- [ ] **Step 3: Add phone worker/channel styles**

Worker panel hidden entirely on phone:
```css
/* In worker-panel.ts */
@media (max-width: 767px) { :host { display: none; } }
```

Channel panel hidden by default (shown in tab):
```css
/* In channel-panel.ts */
@media (max-width: 767px) {
  :host { display: none; }
  :host(.phone-tab-active) { display: flex; position: static; width: 100%; height: 100%; }
}
```

- [ ] **Step 4: Restructure render method for phone**

Change `render()` to wrap terminal and channel panel in `.tab-panel` containers. Add tab bar. Worker panel excluded from phone render:

```typescript
override render() {
  return html`
    <claudony-worker-panel></claudony-worker-panel>
    <div class="tab-content">
      <div class="tab-panel ${this._activeTab === 'terminal' ? 'active' : ''}">
        <div id="terminal-container"></div>
      </div>
      <div class="tab-panel ${this._activeTab === 'chat' ? 'active' : ''}">
        <claudony-channel-panel></claudony-channel-panel>
      </div>
    </div>
    <nav class="tab-bar" role="tablist" aria-label="Panel navigation">
      <button class="tab-btn" role="tab" aria-selected=${this._activeTab === 'terminal'}
        @click=${() => this._switchTab('terminal')}>Terminal</button>
      <button class="tab-btn" role="tab" aria-selected=${this._activeTab === 'chat'}
        @click=${() => this._switchTab('chat')}>Chat</button>
    </nav>
  `;
}
```

The `.tab-content` wrapper and `.tab-bar` are always in the DOM. On desktop/tablet, `.tab-content` is a normal flex container and `.tab-bar` is `display: none`. On phone, the media query activates the absolute positioning and shows the tab bar.

- [ ] **Step 5: Visual check — standalone session at phone viewport**

Open a standalone session at 375x812:
- Terminal tab fills screen (default)
- Chat tab shows channel panel full-screen
- Tab bar at bottom with 2 tabs
- Worker panel not visible (no tab)
- Terminal state preserved on tab switch

- [ ] **Step 6: Commit**

```
feat(#179): terminal-workspace phone — tab bar + full-screen panels
```

---

### Task 8: Key-bar responsive

**Files:**
- Modify: `app/src/main/webui/src/components/key-bar.ts` (Shadow DOM styles + `active-tab-changed` listener)

**Interfaces:**
- Consumes: `active-tab-changed` pages-event from Tasks 5 and 7
- Produces: 44px touch targets on tablet; hidden on non-terminal phone tabs

- [ ] **Step 1: Add `_terminalActive` state and event listener**

Use `ide_insert_member`:

```typescript
@state() private _terminalActive = true;
```

Update `connectedCallback()` — add event listener:

```typescript
connectedCallback(): void {
  super.connectedCallback();
  this._isTouch = 'ontouchstart' in window || navigator.maxTouchPoints > 0;
  if (!this._isTouch) this.setAttribute('hidden', '');
  document.addEventListener('pages-event', this._onTabChanged as EventListener);
}

disconnectedCallback(): void {
  super.disconnectedCallback();
  document.removeEventListener('pages-event', this._onTabChanged as EventListener);
}

private _onTabChanged = ((e: CustomEvent) => {
  if (e.detail?.topic === 'active-tab-changed') {
    this._terminalActive = e.detail.payload.tab === 'terminal';
  }
}) as EventListener;
```

- [ ] **Step 2: Update render to conditionally hide on non-terminal tabs**

```typescript
override render() {
  if (!this._isTouch || !this._terminalActive) return nothing;
  // ... existing render
}
```

- [ ] **Step 3: Add touch target sizing media query**

Append to Shadow DOM styles:

```css
@media (max-width: 1024px) {
  pages-button { min-height: 44px; min-width: 44px; }
  .key-bar { padding: 4px 8px; }
}
```

- [ ] **Step 4: Commit**

```
feat(#179): key-bar responsive — touch targets + tab visibility
```

---

### Task 9: E2E viewport tests

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/e2e/ResponsiveLayoutE2ETest.java`

**Interfaces:**
- Consumes: all responsive CSS from Tasks 1-8
- Produces: E2E test coverage for critical responsive paths

- [ ] **Step 1: Create viewport-parameterised E2E test class**

```java
package io.casehub.claudony.e2e;

import com.microsoft.playwright.Page;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
class ResponsiveLayoutE2ETest extends PlaywrightBase {

    @Test
    void fleetHome_phone_singleColumnGrid() {
        page.setViewportSize(375, 812);
        page.navigate(BASE_URL + "/app/");
        // Assert session grid is single column
        var grid = page.locator("#session-grid");
        assertThat(grid).isVisible();
        // Fleet panel hidden at phone breakpoint
        var fleet = page.locator("#fleet-panel");
        assertThat(fleet).isHidden();
    }

    @Test
    void fleetHome_tablet_gridVisible() {
        page.setViewportSize(768, 1024);
        page.navigate(BASE_URL + "/app/");
        var grid = page.locator("#session-grid");
        assertThat(grid).isVisible();
    }
}
```

- [ ] **Step 2: Run E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=ResponsiveLayoutE2ETest
```

- [ ] **Step 3: Commit**

```
test(#179): E2E viewport tests for responsive layouts
```

---

## Execution Order

```
Task 1 (foundation)
  ├── Task 2 (fleet home)
  ├── Task 3 (terminal header)
  ├── Task 4 (workbench tablet)
  │   └── Task 5 (workbench phone)
  ├── Task 6 (terminal-workspace tablet)
  │   └── Task 7 (terminal-workspace phone)
  └── Task 8 (key-bar) — after Tasks 5 + 7
Task 9 (E2E) — after all above
```

Tasks 2, 3, 4, and 6 can execute in parallel after Task 1.
Tasks 5 and 7 depend on their tablet predecessors.
Task 8 depends on the phone layouts (for the tab-changed event).
Task 9 depends on all.
