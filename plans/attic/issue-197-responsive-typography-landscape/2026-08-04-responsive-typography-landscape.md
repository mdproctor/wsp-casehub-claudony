# Responsive Typography + Landscape Phone Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #197 — responsive typography scaling
**Issue group:** #197, #196

**Goal:** Replace all hardcoded font sizes with `--pages-font-size-*` design tokens, add breakpoint-based scaling, and optimise the terminal page for phone landscape orientation.

**Architecture:** Token overrides live in `THEME_CSS` (shadow DOM components) and `style.css` (global light DOM). Landscape phone mode uses a CSS media query to hide chrome and show a floating nav overlay. No new dependencies.

**Tech Stack:** Lit 3, CSS custom properties, `@casehubio/pages-ui-tokens`

## Global Constraints

- Token values come from `pages-ui-tokens/src/tokens.ts` — xs=10px, sm=11px, base=13px, lg=14px, xl=18px, 2xl=20px
- Two breakpoints for token scaling: ≤767px (phone) and ≥1440px (large screen)
- Landscape trigger: `@media (orientation: landscape) and (max-height: 500px)`
- WCAG minimum touch target: 44×44px
- All CSS changes must respect `prefers-reduced-motion`

---

### Task 1: Add Breakpoint Token Overrides to THEME_CSS and style.css

**Files:**
- Modify: `app/src/main/webui/src/theme.ts`
- Modify: `app/src/main/resources/META-INF/resources/app/style.css`

**Interfaces:**
- Consumes: nothing
- Produces: `THEME_CSS` constant with responsive token overrides (consumed by all subsequent tasks)

- [ ] **Step 1: Add token overrides to THEME_CSS in theme.ts**

In `app/src/main/webui/src/theme.ts`, add the breakpoint media queries inside the `THEME_CSS` template literal, after the existing `:host` block:

```typescript
export const THEME_CSS = `
  :host {
    --bg: var(--pages-neutral-1);
    --surface: var(--pages-neutral-2);
    --border: var(--pages-neutral-4);
    --text: var(--pages-neutral-11);
    --text-muted: var(--pages-neutral-8);
    --accent: var(--pages-accent-9);
    --active: var(--pages-success-9);
    --danger: var(--pages-danger-9);
    --radius: var(--pages-radius-md);
    font-family: var(--pages-font-family);
    color: var(--text);
  }
  @media (max-width: 767px) {
    :host {
      --pages-font-size-xs: 9px;
      --pages-font-size-sm: 10px;
      --pages-font-size-base: 12px;
      --pages-font-size-lg: 13px;
      --pages-font-size-xl: 16px;
      --pages-font-size-2xl: 18px;
    }
  }
  @media (min-width: 1440px) {
    :host {
      --pages-font-size-xs: 11px;
      --pages-font-size-sm: 12px;
      --pages-font-size-base: 14px;
      --pages-font-size-lg: 15px;
      --pages-font-size-xl: 20px;
      --pages-font-size-2xl: 22px;
    }
  }
`;
```

- [ ] **Step 2: Add matching overrides to style.css**

In `app/src/main/resources/META-INF/resources/app/style.css`, add after the existing `@media (max-width: 767px)` blocks (before the Fleet layout section):

```css
@media (max-width: 767px) {
  :root {
    --pages-font-size-xs: 9px;
    --pages-font-size-sm: 10px;
    --pages-font-size-base: 12px;
    --pages-font-size-lg: 13px;
    --pages-font-size-xl: 16px;
    --pages-font-size-2xl: 18px;
  }
}
@media (min-width: 1440px) {
  :root {
    --pages-font-size-xs: 11px;
    --pages-font-size-sm: 12px;
    --pages-font-size-base: 14px;
    --pages-font-size-lg: 15px;
    --pages-font-size-xl: 20px;
    --pages-font-size-2xl: 22px;
  }
}
```

- [ ] **Step 3: Run vitest to verify no breakage**

Run: `npm --prefix app/src/main/webui test`
Expected: all existing tests pass (no font-size tests exist)

- [ ] **Step 4: Commit**

```bash
git add app/src/main/webui/src/theme.ts app/src/main/resources/META-INF/resources/app/style.css
git commit -m "feat(ui): add responsive typography token overrides at 767px and 1440px

Refs #197"
```

---

### Task 2: Migrate style.css to Design Tokens

**Files:**
- Modify: `app/src/main/resources/META-INF/resources/app/style.css`

**Interfaces:**
- Consumes: token overrides from Task 1
- Produces: style.css with all font-size values expressed as tokens

Replace every hardcoded `font-size` in `style.css` with the appropriate `var(--pages-font-size-*)`. This is the largest single file — ~45 replacements.

- [ ] **Step 1: Replace all font-size declarations**

Apply these replacements throughout `style.css`:

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 37 | `h1` | `18px` | `var(--pages-font-size-xl)` |
| 45 | `button` | `14px` | `var(--pages-font-size-lg)` |
| 76 | `.card-name` | `15px` | `var(--pages-font-size-xl)` |
| 79 | `.badge` | `11px` | `var(--pages-font-size-sm)` |
| 87 | `.card-dir` | `12px` | `var(--pages-font-size-base)` |
| 88 | `.card-meta` | `11px` | `var(--pages-font-size-sm)` |
| 90 | `.card-actions button` | `12px` | `var(--pages-font-size-base)` |
| 97 | `.card-git` | `12px` | `var(--pages-font-size-base)` |
| 98 | `.card-services` | `12px` | `var(--pages-font-size-base)` |
| 99 | `.svc-label` | `11px` | `var(--pages-font-size-sm)` |
| 120 | `.empty-state p` | `15px` | `var(--pages-font-size-xl)` |
| 128 | `dialog h2` | `16px` | `var(--pages-font-size-xl)` |
| 129 | `dialog label` | `13px` | `var(--pages-font-size-base)` |
| 134 | `dialog input` | `14px` | `var(--pages-font-size-lg)` |
| 154 | `.compose-header span:first-child` | `14px` | `var(--pages-font-size-lg)` |
| 155 | `.compose-hint` | `11px` | `var(--pages-font-size-sm)` |
| 159 | `.compose-textarea` | `14px` | `var(--pages-font-size-lg)` |
| 208 | `.fleet-panel-title` | `12px` | `var(--pages-font-size-base)` |
| 216 | `button.small` | `11px` | `var(--pages-font-size-sm)` |
| 222 | `.peer-empty` | `12px` | `var(--pages-font-size-base)` |
| 232 | `.peer-card` | `12px` | `var(--pages-font-size-base)` |
| 244 | `.peer-name` | `12px` | `var(--pages-font-size-base)` |
| 252 | `.peer-source` | `10px` | `var(--pages-font-size-xs)` |
| 263 | `.peer-url` | `10px` | `var(--pages-font-size-xs)` |
| 276 | `.peer-meta` | `11px` | `var(--pages-font-size-sm)` |
| 286 | `.peer-actions button` | `10px` | `var(--pages-font-size-xs)` |
| 304 | `.circuit-label` | `10px` | `var(--pages-font-size-xs)` |
| 318 | `.instance-badge` | `10px` | `var(--pages-font-size-xs)` |
| 339 | `.stale-badge` | `10px` | `var(--pages-font-size-xs)` |
| 358 | `dialog select` | `14px` | `var(--pages-font-size-lg)` |
| 399 | `.mesh-title` | `11px` | `var(--pages-font-size-sm)` |
| 411 | `.mesh-view-btn` | `12px` | `var(--pages-font-size-base)` |
| 423 | `.mesh-collapse-btn` | `13px` | `var(--pages-font-size-base)` |
| 446 | `.mesh-expand` | `10px` | `var(--pages-font-size-xs)` |
| 456 | `.mesh-empty` | `13px` | `var(--pages-font-size-base)` |
| 462 | `.mesh-label` | `10px` | `var(--pages-font-size-xs)` |
| 474 | `.mesh-instance` | `11px` | `var(--pages-font-size-sm)` |
| 486 | `.mesh-channel-name` | `12px` | `var(--pages-font-size-base)` |
| 488 | `.mesh-channel-count` | `10px` | `var(--pages-font-size-xs)` |
| 494 | `.mesh-msg` | `11px` | `var(--pages-font-size-sm)` |
| 497 | `.mesh-dim` | `12px` | `var(--pages-font-size-base)` |
| 506 | `.mesh-channel-select` | `12px` | `var(--pages-font-size-base)` |
| 512 | `.mesh-feed-item` | `11px` | `var(--pages-font-size-sm)` |
| 521 | `.mesh-channel-tag` | `10px` | `var(--pages-font-size-xs)` |
| 556 | `.mesh-dock-controls select` | `0.75rem` | `var(--pages-font-size-sm)` |
| 567 | `.mesh-dock-textarea` | `0.8rem` | `var(--pages-font-size-base)` |
| 579 | `.mesh-dock-send` | `0.75rem` | `var(--pages-font-size-sm)` |
| 591 | `.mesh-dock-error` | `0.7rem` | `var(--pages-font-size-sm)` |

- [ ] **Step 2: Run vitest**

Run: `npm --prefix app/src/main/webui test`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add app/src/main/resources/META-INF/resources/app/style.css
git commit -m "feat(ui): migrate style.css font sizes to design tokens

Refs #197"
```

---

### Task 3: Migrate Shadow-DOM Components to Design Tokens

**Files:**
- Modify: `app/src/main/webui/src/components/session-panel.ts`
- Modify: `app/src/main/webui/src/components/terminal-header.ts`
- Modify: `app/src/main/webui/src/components/terminal-workspace.ts`
- Modify: `app/src/main/webui/src/components/worker-panel.ts`
- Modify: `app/src/main/webui/src/components/channel-panel.ts`
- Modify: `app/src/main/webui/src/components/claudony-fleet-panel.ts`
- Modify: `app/src/main/webui/src/components/claudony-workbench.ts`
- Modify: `app/src/main/webui/src/components/claudony-mesh-panel.ts`
- Modify: `app/src/main/webui/src/components/claudony-action-inbox.ts`

**Interfaces:**
- Consumes: token overrides from Task 1
- Produces: all components using tokens instead of hardcoded values

Replace every hardcoded `font-size` in each component's `static styles` CSS block with `var(--pages-font-size-*)`.

- [ ] **Step 1: Migrate session-panel.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 55 | `.header h2` | `1.1rem` | `var(--pages-font-size-lg)` |
| 63 | `.card-name` | `0.95rem` | `var(--pages-font-size-base)` |
| 64 | `.card-dir` | `0.8rem` | `var(--pages-font-size-base)` |
| 65 | `.card-time` | `0.75rem` | `var(--pages-font-size-sm)` |
| 67 | `.card-git, .card-services` | `12px` | `var(--pages-font-size-base)` |
| 72 | `.auth-body p` | `0.9rem` | `var(--pages-font-size-base)` |
| 77 | `.view-toggle button` | `14px` | `var(--pages-font-size-lg)` |

- [ ] **Step 2: Migrate terminal-header.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 22 | `.back-link` | `14px` | `var(--pages-font-size-lg)` |
| 26 | `.session-name` | `14px` | `var(--pages-font-size-lg)` |

For the media query overrides — preserve the 3-tier scaling using tokens:

```css
@media (max-width: 1024px) {
  .back-link::before { font-size: var(--pages-font-size-xl); }
  .session-name { font-size: var(--pages-font-size-base); }
}
@media (max-width: 767px) {
  .session-name { font-size: var(--pages-font-size-base); max-width: 40vw; }
}
```

Note: at 1024px, session-name drops from `--pages-font-size-lg` (14px) to `--pages-font-size-base` (13px). At 767px, `--pages-font-size-base` is overridden to 12px by the global token override from Task 1, achieving the original 3-step scaling (14→13→12).

- [ ] **Step 3: Migrate terminal-workspace.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 51 | `.tab-btn` | `12px` | `var(--pages-font-size-base)` |

- [ ] **Step 4: Migrate worker-panel.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 34 | `.panel-title` | `12px` | `var(--pages-font-size-base)` |
| 40 | `.worker-row` | `13px` | `var(--pages-font-size-base)` |
| 47 | `.worker-time` | `11px` | `var(--pages-font-size-sm)` |
| 50 | `.placeholder` | `13px` | `var(--pages-font-size-base)` |

- [ ] **Step 5: Migrate channel-panel.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 85 | `.ch-select` | `12px` | `var(--pages-font-size-base)` |
| 91 | tab buttons | `13px` | `var(--pages-font-size-base)` |
| 102 | `.case-role` | `12px` | `var(--pages-font-size-base)` |
| 111 | `.case-elapsed` | `11px` | `var(--pages-font-size-sm)` |
| 115 | `.lineage-toggle` | `11px` | `var(--pages-font-size-sm)` |
| 119 | `.chevron` | `9px` | `var(--pages-font-size-xs)` |
| 128 | `.lineage-row` | `11px` | `var(--pages-font-size-sm)` |
| 140 | `.lineage-empty` | `11px` | `var(--pages-font-size-sm)` |
| 147 | `.stale-prompt` | `12px` | `var(--pages-font-size-base)` |
| 152 | `.stale-btn` | `11px` | `var(--pages-font-size-sm)` |
| 158 | `.error` | `11px` | `var(--pages-font-size-sm)` |

- [ ] **Step 6: Migrate claudony-fleet-panel.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 49 | `.title` | `12px` | `var(--pages-font-size-base)` |
| 53 | `.peer-empty` | `12px` | `var(--pages-font-size-base)` |
| 56 | `.peer-card` | `12px` | `var(--pages-font-size-base)` |
| 60 | `.peer-name` | `12px` | `var(--pages-font-size-base)` |
| 64 | `.peer-source` | `10px` | `var(--pages-font-size-xs)` |
| 69 | `.peer-url` | `10px` | `var(--pages-font-size-xs)` |
| 74 | `.peer-meta` | `11px` | `var(--pages-font-size-sm)` |

- [ ] **Step 7: Migrate claudony-workbench.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 158 | `.dock-btn` | `11px` | `var(--pages-font-size-sm)` |
| 182 | `.case-role` | `12px` | `var(--pages-font-size-base)` |
| 190 | `.case-elapsed` | `11px` | `var(--pages-font-size-sm)` |
| 194 | `.lineage-toggle` | `11px` | `var(--pages-font-size-sm)` |
| 198 | `.chevron` | `9px` | `var(--pages-font-size-xs)` |
| 207 | `.lineage-row` | `11px` | `var(--pages-font-size-sm)` |
| 219 | `.lineage-empty` | `11px` | `var(--pages-font-size-sm)` |
| 226 | `.stale-prompt` | `12px` | `var(--pages-font-size-base)` |
| 231 | `.stale-btn` | `11px` | `var(--pages-font-size-sm)` |
| 237 | `.error` | `11px` | `var(--pages-font-size-sm)` |
| 245 | `.worker-row` | `13px` | `var(--pages-font-size-base)` |
| 257 | `.worker-time` | `11px` | `var(--pages-font-size-sm)` |
| 260 | `.section-title` | `11px` | `var(--pages-font-size-sm)` |

- [ ] **Step 8: Migrate claudony-mesh-panel.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 60 | `.title` | `11px` | `var(--pages-font-size-sm)` |
| 67 | `.label` | `10px` | `var(--pages-font-size-xs)` |
| 74 | `.channel-name` | `12px` | `var(--pages-font-size-base)` |
| 76 | `.channel-count` | `10px` | `var(--pages-font-size-xs)` |
| 79 | `.msg` | `11px` | `var(--pages-font-size-sm)` |
| 84 | `.feed-item` | `11px` | `var(--pages-font-size-sm)` |
| 86 | `.feed-tag` | `10px` | `var(--pages-font-size-xs)` |
| 111 | `.create-form input` | `12px` | `var(--pages-font-size-base)` |
| 117 | `.create-form .create-error` | `10px` | `var(--pages-font-size-xs)` |
| 122 | `.ch-select` | `12px` | `var(--pages-font-size-base)` |
| 147 | `.stale-prompt` | `12px` | `var(--pages-font-size-base)` |
| 152 | `.stale-btn` | `11px` | `var(--pages-font-size-sm)` |

- [ ] **Step 9: Migrate claudony-action-inbox.ts**

| Line | Selector | Old | New |
|------|----------|-----|-----|
| 43 | `.summary` | `0.875rem` | `var(--pages-font-size-base)` |
| 46 | `table` | `0.8125rem` | `var(--pages-font-size-base)` |
| 55 | `.actions button` | `0.75rem` | `var(--pages-font-size-sm)` |

- [ ] **Step 10: Run vitest**

Run: `npm --prefix app/src/main/webui test`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add app/src/main/webui/src/components/
git commit -m "feat(ui): migrate all component font sizes to design tokens

Refs #197"
```

---

### Task 4: Landscape Phone Optimisation (#196)

**Files:**
- Modify: `app/src/main/webui/src/components/terminal-header.ts`
- Modify: `app/src/main/webui/src/components/terminal-workspace.ts`

**Interfaces:**
- Consumes: token overrides from Task 1 (for overlay button font sizes)
- Produces: phone-landscape mode with hidden chrome and floating nav overlay

- [ ] **Step 1: Add landscape hide rule to terminal-header.ts**

Add to the component's `static styles`:

```css
@media (orientation: landscape) and (max-height: 500px) {
  :host { display: none !important; }
}
```

- [ ] **Step 2: Add landscape styles and overlay to terminal-workspace.ts**

Add the landscape media query to the `static styles`:

```css
@media (orientation: landscape) and (max-height: 500px) {
  .tab-bar { display: none !important; }
  .landscape-nav {
    display: flex;
    position: fixed;
    top: env(safe-area-inset-top, 0);
    left: env(safe-area-inset-left, 0);
    z-index: 100;
    gap: 4px;
    padding: 4px;
    pointer-events: none;
  }
  .landscape-nav button {
    pointer-events: auto;
    width: 44px;
    height: 44px;
    border-radius: 50%;
    border: none;
    background: rgba(30, 30, 30, 0.7);
    color: var(--pages-neutral-11, #ccc);
    font-size: var(--pages-font-size-xl);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }
  .landscape-nav button:hover {
    background: rgba(30, 30, 30, 0.9);
  }
}
```

- [ ] **Step 3: Add landscape-nav to render template**

In the `render()` method of `ClaudonyTerminalWorkspace`, add the overlay markup. It is always in the DOM but only displayed via the landscape media query:

```html
<div class="landscape-nav">
  <button @click=${this._navigateBack} aria-label="Back to sessions">←</button>
  <button @click=${this._toggleLandscapeTab} aria-label="Toggle chat">💬</button>
</div>
```

- [ ] **Step 4: Add the handler methods**

```typescript
private _navigateBack(): void {
  window.location.href = '/app/';
}

private _toggleLandscapeTab(): void {
  this._activeTab = this._activeTab === 'terminal' ? 'chat' : 'terminal';
}
```

- [ ] **Step 5: Write Playwright E2E test for landscape mode**

Add a test to the existing E2E test file (or create `LandscapeE2ETest.java` if a new file is cleaner). The test sets a phone-landscape viewport and verifies:

```java
@Test
void landscapePhone_hidesHeaderAndTabBar_showsFloatingNav() {
    page.setViewportSize(667, 375);
    page.navigate(BASE_URL + "/app/session.html?id=" + testSessionId);

    // Header should be hidden
    assertThat(page.locator("claudony-terminal-header")).not().isVisible();

    // Tab bar should be hidden
    assertThat(page.locator(".tab-bar")).not().isVisible();

    // Floating nav should be visible
    assertThat(page.locator(".landscape-nav")).isVisible();
    assertThat(page.locator(".landscape-nav button")).hasCount(2);
}
```

- [ ] **Step 6: Run E2E test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest`
Expected: PASS (existing + new test)

- [ ] **Step 7: Commit**

```bash
git add app/src/main/webui/src/components/terminal-header.ts app/src/main/webui/src/components/terminal-workspace.ts
git commit -m "feat(ui): landscape phone optimisation — hide chrome, add floating nav

Refs #196"
```

---

### Task 5: Visual Verification and Final Commit

**Files:**
- No new files

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified, working responsive typography and landscape mode

- [ ] **Step 1: Run full vitest suite**

Run: `npm --prefix app/src/main/webui test`
Expected: all tests PASS

- [ ] **Step 2: Run full Playwright E2E suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest`
Expected: all tests PASS

- [ ] **Step 3: Start dev server and visually verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server`

Check in browser:
- Default viewport (1024-1440px): fonts at token defaults
- Phone viewport (375px width): fonts scaled down per 767px breakpoint
- Large viewport (1440px+): fonts scaled up per 1440px breakpoint
- Phone landscape (667×375): terminal header hidden, tab bar hidden, floating nav visible with back + chat buttons
- Tablet landscape (1024×768): no landscape changes (above 500px height threshold)

- [ ] **Step 4: Verify no remaining hardcoded font-size values**

Search all component files for any remaining hardcoded font-size that should have been migrated:

```bash
grep -rn "font-size:" app/src/main/webui/src/components/ app/src/main/resources/META-INF/resources/app/style.css | grep -v "var(--pages-font-size" | grep -v ".casehub-packages" | grep -v "node_modules"
```

Any hits should be either intentional (e.g. the `font-size: 0` reset in terminal-header back-link) or inside media queries that reference tokens.
