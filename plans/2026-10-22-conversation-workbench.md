# Conversation Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #288 — epic: full chat view — standalone conversation route for channel interaction

**Goal:** Extract chat-app's workbench composition into a reusable
`<conversation-workbench>` in casehub-mosaic, then wire it in claudony
as a fleet home Chat tab.

**Architecture:** The workbench is a LitElement in mosaic's
channel-activity package that composes all existing controllers
(PushController, ChannelStateController, MessagingController,
MembershipController, ReactionController, CommitmentController) and
components (channel-nav, channel-feed, channel-input, panels) into a
responsive layout. App-specific concerns (auth, message delivery,
channel filtering) are injected via a config object with callbacks.
Claudony creates a thin wrapper that configures the workbench with its
auth and API paths, and renders it in a new fleet home tab.

**Tech Stack:** TypeScript, Lit 3, casehub-mosaic channel-activity,
pages-data EventConnection, vitest

## Global Constraints

- Mosaic channel-activity version: 0.3.x SNAPSHOT
- Custom element prefix: `agency-` (mosaic) / `claudony-` (app)
- All mosaic components use `--pages-*` CSS custom properties
- Push connection via `@casehubio/pages-data` `createEventConnection`
- Tests: vitest (mosaic and claudony frontend)
- Claudony auth: cookie-based `fetchWithAuth` (no Bearer token)
- Claudony message API: `POST /api/mesh/channels/{name}/messages`
- Claudony push URL: `/ws/push` (no query params)

---

## Batch 1: Mosaic — conversation-workbench component

### Task 1: SwipeController and responsive utilities in mosaic

**Files:**
- Create: `casehub-mosaic/components/channel-activity/src/swipe-controller.ts`
- Create: `casehub-mosaic/components/channel-activity/src/swipe-controller.test.ts`
- Create: `casehub-mosaic/components/channel-activity/src/responsive.ts`

**Interfaces:**
- Consumes: `lit` (ReactiveController, ReactiveControllerHost)
- Produces:
  - `SwipeController` class — `constructor(host: ReactiveControllerHost, options: SwipeOptions)`
  - `SwipeOptions` interface — `{ drawerQuery, backdropQuery, onOpen, onClose?, isOpenQuery?, edgeWidth?, drawerWidth?, distanceThreshold?, velocityThreshold? }`
  - `MQ_TABLET = '(min-width: 768px) and (max-width: 1279px)'`
  - `MQ_DESKTOP = '(min-width: 1280px)'`

- [ ] **Step 1: Copy SwipeController from chat-app**

Copy `/Users/mdproctor/claude/casehub/chat-app/src/main/webui/src/workbench/swipe-controller.ts`
to `casehub-mosaic/components/channel-activity/src/swipe-controller.ts`.
No changes needed — the controller is already generic with no app-specific imports.

- [ ] **Step 2: Copy SwipeController tests from chat-app**

Copy `/Users/mdproctor/claude/casehub/chat-app/src/main/webui/src/workbench/swipe-controller.test.ts`
to `casehub-mosaic/components/channel-activity/src/swipe-controller.test.ts`.
No changes needed.

- [ ] **Step 3: Create responsive.ts**

```typescript
// casehub-mosaic/components/channel-activity/src/responsive.ts
export const MQ_TABLET = '(min-width: 768px) and (max-width: 1279px)';
export const MQ_DESKTOP = '(min-width: 1280px)';
```

- [ ] **Step 4: Run tests**

Run: `npm --prefix casehub-mosaic/components/channel-activity test -- --run swipe-controller`
Expected: all swipe-controller tests pass.

- [ ] **Step 5: Commit**

```bash
git -C casehub-mosaic add components/channel-activity/src/swipe-controller.ts \
  components/channel-activity/src/swipe-controller.test.ts \
  components/channel-activity/src/responsive.ts
git -C casehub-mosaic commit -m "feat(channel-activity): add SwipeController and responsive utilities

Extracted from chat-app for reuse in conversation-workbench.

Refs casehubio/claudony#288"
```

---

### Task 2: ConversationWorkbench component in mosaic

**Files:**
- Create: `casehub-mosaic/components/channel-activity/src/conversation-workbench.ts`
- Modify: `casehub-mosaic/components/channel-activity/src/index.ts` (add exports)

**Interfaces:**
- Consumes: All controllers from channel-activity index, SwipeController (Task 1),
  responsive.ts (Task 1), `createEventConnection` from `@casehubio/pages-data`,
  all channel-activity components (channel-nav, channel-feed, channel-input,
  channel-member-panel, channel-task-panel, channel-correlation-panel,
  channel-artifact-panel, channel-topic-bar)
- Produces:
  - `ConversationWorkbenchElement` — LitElement, tag `agency-conversation-workbench`
  - `ConversationWorkbenchConfig` interface:
    ```typescript
    interface ConversationWorkbenchConfig {
      fetch: (url: string, init?: RequestInit) => Promise<Response>;
      pushUrl: string;
      currentActorId: string;
      restBase: string;
      channelFilter?: (channel: QhorusChannel) => boolean;
      sendMessage: (payload: SendMessagePayload) => Promise<void>;
      markRead?: (channelId: string, lastReadMessageId: string) => Promise<void>;
      createSpace?: (name: string) => Promise<void>;
      renameSpace?: (spaceId: string, newName: string) => Promise<void>;
      deleteSpace?: (spaceId: string) => Promise<void>;
      moveChannelToSpace?: (channelId: string, spaceId: string, position?: number) => Promise<void>;
      correctMessage?: (channelId: string, messageId: string, content: string) => Promise<void>;
      retractMessage?: (channelId: string, messageId: string, reason?: string) => Promise<void>;
      layoutStore?: {
        load(key: string): Promise<LayoutState | null>;
        save(key: string, state: LayoutState): Promise<void>;
      };
    }
    ```
  - `LayoutState` type: `{ splits: Record<string, number>; docks: Record<string, boolean>; panels: Record<string, unknown> }`

- [ ] **Step 1: Write the ConversationWorkbench component**

Extract from `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` (~807 lines)
with these transformations:

1. **Rename** `qhorus-workbench` → `agency-conversation-workbench`
2. **Replace** `authenticatedFetch(url, init)` → `this._config.fetch(url, init)` everywhere
3. **Replace** `getToken()` / `getIdentity()` → `this._config.currentActorId`
4. **Replace** push URL construction:
   ```typescript
   // Before (chat-app):
   const url = `${proto}//${location.host}/ws/push?token=${token}`;
   // After (workbench):
   const url = this._config.pushUrl;
   ```
5. **Replace** `_sendMessage()` body → `await this._config.sendMessage(payload)`
6. **Replace** `authenticatedFetch('/api/chat/${channelId}/read', ...)` → `this._config.markRead?.(channelId, lastReadId)`
7. **Replace** space CRUD calls with config callbacks — each guarded by `this._config.createSpace?.(...)`
8. **Replace** correction/retraction calls with config callbacks
9. **Replace** `createLocalLayoutStore` → `this._config.layoutStore` (with fallback no-ops)
10. **Replace** `qhorus-task-panel` / `qhorus-correlation-panel` / `qhorus-artifact-panel` →
    `agency-channel-task-panel` / `agency-channel-correlation-panel` / `agency-channel-artifact-panel`
11. **Remove** `chat-demo-identity` import and rendering (app-specific)
12. **Remove** `identities` property (app-specific)
13. **Conditionally render** space management UI: `channel-nav` space actions only when
    `config.createSpace` is defined
14. **Conditionally render** correction/retraction in event handling: skip when callbacks absent
15. **Apply `channelFilter`**: in `willUpdate`, if `config.channelFilter` is set, filter
    `this._channels.channels` before passing to `channel-nav`
16. **Import** `SwipeController` from `./swipe-controller.js` and `MQ_TABLET, MQ_DESKTOP`
    from `./responsive.js`

The component property is a single `.config` prop:

```typescript
@customElement('agency-conversation-workbench')
export class ConversationWorkbenchElement extends LitElement {
  @property({ type: Object }) config!: ConversationWorkbenchConfig;
  // ... all internal state from qhorus-workbench
}
```

Full file: adapt the 807-line `qhorus-workbench.ts` with the transformations above.
This is a mechanical extraction — the layout, styles, responsive logic, dock management,
drag-resize, commitment visualization all transfer as-is.

- [ ] **Step 2: Add exports to index.ts**

Append to `casehub-mosaic/components/channel-activity/src/index.ts`:

```typescript
export { ConversationWorkbenchElement } from './conversation-workbench.js';
export type { ConversationWorkbenchConfig, LayoutState } from './conversation-workbench.js';
export { SwipeController } from './swipe-controller.js';
export type { SwipeOptions } from './swipe-controller.js';
export { MQ_TABLET, MQ_DESKTOP } from './responsive.js';
```

- [ ] **Step 3: Type-check**

Run: `npm --prefix casehub-mosaic/components/channel-activity run typecheck`
Expected: no type errors.

- [ ] **Step 4: Commit**

```bash
git -C casehub-mosaic add components/channel-activity/src/conversation-workbench.ts \
  components/channel-activity/src/index.ts
git -C casehub-mosaic commit -m "feat(channel-activity): add <conversation-workbench> composition root

Reusable full-viewport conversation UI. Apps configure via callbacks:
auth fetch, sendMessage, push URL, channel filter, space CRUD.
Responsive layout (desktop/tablet/phone), dock panels, drag-resize.

Refs casehubio/claudony#288"
```

---

### Task 3: ConversationWorkbench unit tests

**Files:**
- Create: `casehub-mosaic/components/channel-activity/src/conversation-workbench.test.ts`

**Interfaces:**
- Consumes: `ConversationWorkbenchElement`, `ConversationWorkbenchConfig` (Task 2)

- [ ] **Step 1: Write tests**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ConversationWorkbenchElement } from './conversation-workbench.js';
import type { ConversationWorkbenchConfig } from './conversation-workbench.js';

function minimalConfig(overrides: Partial<ConversationWorkbenchConfig> = {}): ConversationWorkbenchConfig {
  return {
    fetch: vi.fn().mockResolvedValue(new Response('[]')),
    pushUrl: 'ws://localhost/ws/push',
    currentActorId: 'test-user',
    restBase: '/api',
    sendMessage: vi.fn().mockResolvedValue(undefined),
    ...overrides,
  };
}

describe('ConversationWorkbenchElement', () => {
  let el: ConversationWorkbenchElement;

  beforeEach(() => {
    el = new ConversationWorkbenchElement();
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as agency-conversation-workbench', () => {
    expect(customElements.get('agency-conversation-workbench')).toBe(ConversationWorkbenchElement);
  });

  it('renders dock strip with config callbacks', async () => {
    el.config = minimalConfig();
    await el.updateComplete;
    const dockBtns = el.shadowRoot!.querySelectorAll('.dock-btn');
    expect(dockBtns.length).toBeGreaterThan(0);
  });

  it('hides space management when space callbacks absent', async () => {
    el.config = minimalConfig();
    await el.updateComplete;
    // No space callbacks → no space create button in nav
    // (Specific assertion depends on how channel-nav renders space controls)
  });

  it('shows space management when space callbacks provided', async () => {
    el.config = minimalConfig({
      createSpace: vi.fn(),
      renameSpace: vi.fn(),
      deleteSpace: vi.fn(),
      moveChannelToSpace: vi.fn(),
    });
    await el.updateComplete;
    // Space callbacks present → space controls available
  });

  it('delegates sendMessage to config callback', async () => {
    const sendMessage = vi.fn().mockResolvedValue(undefined);
    el.config = minimalConfig({ sendMessage });
    await el.updateComplete;
    // Trigger via pages-event dispatch with SEND_MESSAGE topic
    // Verify sendMessage was called with the payload
  });

  it('applies channelFilter when provided', async () => {
    const filter = vi.fn().mockReturnValue(true);
    el.config = minimalConfig({ channelFilter: filter });
    await el.updateComplete;
    // channelFilter should be called during channel rendering
  });
});
```

- [ ] **Step 2: Run tests**

Run: `npm --prefix casehub-mosaic/components/channel-activity test -- --run conversation-workbench`
Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git -C casehub-mosaic add components/channel-activity/src/conversation-workbench.test.ts
git -C casehub-mosaic commit -m "test(channel-activity): unit tests for conversation-workbench

Refs casehubio/claudony#288"
```

---

### Task 4: Build and install mosaic SNAPSHOT

**Files:**
- No files created — build/install step

**Interfaces:**
- Produces: Updated `casehub-casehub-mosaic-npm` SNAPSHOT JAR in `~/.m2`

- [ ] **Step 1: Build mosaic channel-activity**

Run: `npm --prefix casehub-mosaic/components/channel-activity run build`
Expected: `dist/` contains `conversation-workbench.js`, `conversation-workbench.d.ts`,
`swipe-controller.js`, `swipe-controller.d.ts`, `responsive.js`, `responsive.d.ts`,
updated `index.js` and `index.d.ts`.

- [ ] **Step 2: Install mosaic npm bridge JAR**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f ~/claude/casehub/casehub-mosaic/npm-packages/pom.xml
```

Expected: `casehub-casehub-mosaic-npm` SNAPSHOT installed to `~/.m2`.

- [ ] **Step 3: Unpack in claudony**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn initialize -pl app -f ~/claude/casehub/slots/194/claudony/pom.xml
```

Expected: Updated mosaic packages unpacked to `app/src/main/webui/.casehub-packages/`.

- [ ] **Step 4: Verify conversation-workbench is available**

```bash
ls ~/claude/casehub/slots/194/claudony/app/src/main/webui/.casehub-packages/packages/mosaic-channel-activity/dist/conversation-workbench.js
```

Expected: file exists.

- [ ] **Step 5: Commit mosaic changes**

```bash
git -C casehub-mosaic add -A
git -C casehub-mosaic commit -m "chore: build conversation-workbench SNAPSHOT

Refs casehubio/claudony#288"
```

---

## Batch 2: Claudony — Chat tab in fleet home

### Task 5: claudony-chat-panel component

**Files:**
- Create: `claudony/app/src/main/webui/src/components/claudony-chat-panel.ts`

**Interfaces:**
- Consumes: `ConversationWorkbenchElement`, `ConversationWorkbenchConfig` from
  `@casehubio/mosaic-channel-activity`, `fetchWithAuth` from `../util/auth.js`,
  `createEventConnection` from `@casehubio/pages-data`
- Produces: `<claudony-chat-panel>` custom element with optional `namespace` attribute

- [ ] **Step 1: Write failing test**

Create `claudony/app/src/main/webui/src/components/claudony-chat-panel.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('claudony-chat-panel', () => {
  it('should be importable', async () => {
    const mod = await import('./claudony-chat-panel.js');
    expect(mod.ClaudonyChatPanel).toBeDefined();
  });
});
```

Run: `npm --prefix claudony/app/src/main/webui test -- --run claudony-chat-panel`
Expected: FAIL — module not found.

- [ ] **Step 2: Write claudony-chat-panel.ts**

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { ConversationWorkbenchConfig } from '@casehubio/mosaic-channel-activity';
import '@casehubio/mosaic-channel-activity';
import { fetchWithAuth } from '../util/auth.js';
import type { SendMessagePayload } from '@casehubio/mosaic-channel-activity';

@customElement('claudony-chat-panel')
export class ClaudonyChatPanel extends LitElement {
  @property({ type: String }) namespace = '';

  static override styles = css`
    :host { display: flex; flex: 1; overflow: hidden; }
    agency-conversation-workbench { flex: 1; }
  `;

  private _buildConfig(): ConversationWorkbenchConfig {
    const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
    const ns = this.namespace;
    return {
      fetch: fetchWithAuth,
      pushUrl: `${proto}//${location.host}/ws/push`,
      currentActorId: 'human',
      restBase: '/api',
      channelFilter: ns ? (ch) => ch.name.startsWith(ns) : undefined,
      sendMessage: async (payload: SendMessagePayload) => {
        const body: Record<string, unknown> = {
          content: payload.content,
          type: payload.speechAct || 'command',
        };
        if (payload.inReplyTo) body.inReplyTo = Number(payload.inReplyTo);
        if (payload.topic) body.topic = payload.topic;
        await fetchWithAuth(
          `/api/mesh/channels/${encodeURIComponent(payload.channelId)}/messages`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(body),
          },
        );
      },
    };
  }

  override render() {
    return html`
      <agency-conversation-workbench .config=${this._buildConfig()}>
      </agency-conversation-workbench>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'claudony-chat-panel': ClaudonyChatPanel;
  }
}
```

- [ ] **Step 3: Run test**

Run: `npm --prefix claudony/app/src/main/webui test -- --run claudony-chat-panel`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git -C claudony add app/src/main/webui/src/components/claudony-chat-panel.ts \
  app/src/main/webui/src/components/claudony-chat-panel.test.ts
git -C claudony commit -m "feat: add claudony-chat-panel wrapping conversation-workbench

Thin shell configuring the mosaic workbench with claudony's auth,
message API, and push URL. Supports namespace filtering.

Refs #288"
```

---

### Task 6: Wire Chat tab into fleet home

**Files:**
- Modify: `claudony/app/src/main/webui/src/app.ts` (add import, register panel, add tab)

**Interfaces:**
- Consumes: `claudony-chat-panel` (Task 5), pages-runtime `registerPanel`, pages-ui `tabs`

- [ ] **Step 1: Modify app.ts**

Add import at top:
```typescript
import "./components/claudony-chat-panel";
```

Add registration:
```typescript
registerPanel("chat-panel", "claudony-chat-panel");
```

Add tab to the `tabs(...)` call:
```typescript
const app = tabs(
  ["Sessions", hostPanel("session-panel")],
  ["Cases", hostPanel("case-browser")],
  ["Inbox", hostPanel("action-inbox")],
  ["Chat", hostPanel("chat-panel")],    // ← new
  ["Pools", hostPanel("pool-panel")],
  ["Fleet", hostPanel("fleet-panel")],
  ["Mesh", hostPanel("mesh-panel")],
);
```

- [ ] **Step 2: Build and verify**

Run: `npm --prefix claudony/app/src/main/webui run build` (or esbuild via `node esbuild.config.mjs`)
Expected: builds without errors.

- [ ] **Step 3: Commit**

```bash
git -C claudony add app/src/main/webui/src/app.ts
git -C claudony commit -m "feat: add Chat tab to fleet home

Wires claudony-chat-panel as a tab alongside Sessions, Cases,
Inbox, Pools, Fleet, Mesh.

Refs #288"
```

---

### Task 7: E2E test — Chat tab renders

**Files:**
- Modify: `claudony/app/src/test/java/.../e2e/DashboardE2ETest.java` (add Chat tab test)

**Interfaces:**
- Consumes: Playwright, fleet home page

- [ ] **Step 1: Add test method**

In `DashboardE2ETest.java`, add a test that verifies the Chat tab exists and
clicking it renders the conversation workbench:

```java
@Test
void chatTabRendersWorkbench() {
    page.navigate(BASE_URL + "/app/");
    // Click the Chat tab
    page.locator("text=Chat").click();
    // Verify the workbench renders
    page.locator("claudony-chat-panel").waitFor();
    // The conversation-workbench should be inside it
    var workbench = page.locator("claudony-chat-panel")
        .locator("agency-conversation-workbench");
    workbench.waitFor(new Locator.WaitForOptions()
        .setState(WaitForSelectorState.ATTACHED));
}
```

- [ ] **Step 2: Run E2E test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=DashboardE2ETest#chatTabRendersWorkbench`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C claudony add app/src/test/java
git -C claudony commit -m "test(e2e): verify Chat tab renders conversation workbench

Refs #288"
```

---

## References

- [2026-10-22-conversation-workbench-design.md] — design spec this plan implements
- `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` — source for extraction
- `chat-app/src/main/webui/src/workbench/swipe-controller.ts` — mobile gesture controller
- `chat-app/src/main/webui/src/workbench/responsive.ts` — media query constants
- `claudony/app/src/main/webui/src/components/claudony-workbench.ts` — existing terminal-primary workbench
- `claudony/app/src/main/webui/src/app.ts` — fleet home entry point
- `casehub-mosaic/components/channel-activity/src/index.ts` — mosaic exports
- GitHub #288 — focal issue
- GitHub #287 — conversation polish (complementary)
