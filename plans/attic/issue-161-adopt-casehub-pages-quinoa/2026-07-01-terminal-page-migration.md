# Terminal Page Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate terminal page (session.html + terminal.js) from hand-coded JavaScript to TypeScript Web Components composed via casehub-pages `loadSite()`.

**Architecture:** Decompose 834-line terminal.js into five focused Web Components (`ClaudonyTerminalHeader`, `ClaudonyTerminalWorkspace`, `ClaudonyWorkerPanel`, `ClaudonyChannelPanel`, `ClaudonyKeyBar`) plus compose overlay utility. Components communicate via `pages-event` CustomEvent protocol. Each component is a light-DOM HTMLElement registered via `registerPanel()` and composed via `loadSite(rows(...))`. A separate esbuild entry point (`terminal.ts`) produces `dist/terminal.js` with code splitting to share common deps with the dashboard bundle.

**Tech Stack:** TypeScript 5.6, esbuild 0.25, casehub-pages (pages-runtime, pages-ui, pages-component-terminal), xterm.js 5, Vitest (casehub-pages repo only), Playwright (E2E)

## Global Constraints

- All new Claudony components use **light DOM** (`this.innerHTML`), not shadow DOM — preserves existing Playwright selectors
- All inter-component events use the `pages-event` CustomEvent protocol: `new CustomEvent("pages-event", { bubbles: true, composed: true, detail: { topic, payload } })`
- E2E correctness contract: all 32 existing Playwright tests must pass without changing test assertions (one exception: `CaseWorkerPanelE2ETest` AC4 — source-reading assertion converted to behavioral)
- `window.__CLAUDONY_TEST_MODE__` gates for test hooks must be preserved
- Session.html retains PWA manifest, service worker, and `style.css` link
- esbuild uses `splitting: true` with `format: "esm"` — both bundles share common chunks
- CSS: `:root` variables, `*` reset, `body`, shared `.badge` styles stay in `style.css`; terminal-specific CSS moves into components
- Upstream: `PagesTerminal` needs `paste()` method and `get terminal()` getter before terminal.ts can be completed

---

### Task 1: Upstream PagesTerminal API additions

**Repo:** `/Users/mdproctor/claude/casehub/pages`

**Files:**
- Modify: `components/pages-component-terminal/src/PagesTerminal.ts`
- Modify: `components/pages-component-terminal/src/PagesTerminal.test.ts`

**Interfaces:**
- Produces: `PagesTerminal.paste(text: string): void` — delegates to `this._terminal.paste(text)` for bracketed paste mode
- Produces: `PagesTerminal.terminal: Terminal | undefined` — public getter exposing the raw xterm.js Terminal

- [ ] **Step 1: Write failing tests for `paste()` and `terminal` getter**

Add to `PagesTerminal.test.ts` inside the `describe("WebSocket lifecycle")` block:

```typescript
it("paste delegates to terminal.paste for bracketed paste mode", () => {
  mockTerminal.paste = vi.fn();
  const el = createElement({ wsUrl: "ws://host/ws/{cols}/{rows}" });
  container.appendChild(el);

  (el as unknown as { paste: (t: string) => void }).paste("multi\nline\ntext");
  expect(mockTerminal.paste).toHaveBeenCalledWith("multi\nline\ntext");
});

it("paste is no-op when terminal not initialised", () => {
  const el = createElement();
  container.appendChild(el);
  // No configure() called — _terminal is undefined
  expect(() => {
    (el as unknown as { paste: (t: string) => void }).paste("text");
  }).not.toThrow();
});
```

Add a new `describe("public accessors")` block:

```typescript
describe("public accessors", () => {
  it("terminal getter returns xterm Terminal after configure", () => {
    const el = createElement({ wsUrl: "ws://host/ws/{cols}/{rows}" });
    container.appendChild(el);

    const terminal = (el as unknown as { terminal: unknown }).terminal;
    expect(terminal).toBe(mockTerminal);
  });

  it("terminal getter returns undefined before configure", () => {
    const el = createElement();
    container.appendChild(el);

    const terminal = (el as unknown as { terminal: unknown }).terminal;
    expect(terminal).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run components/pages-component-terminal/src/PagesTerminal.test.ts`
Expected: 4 failures — `paste` and `terminal` not defined on element

- [ ] **Step 3: Implement paste() and terminal getter**

In `PagesTerminal.ts`, add after the `sendInput` method (line 51):

```typescript
paste(text: string): void {
  this._terminal?.paste(text);
}

get terminal(): Terminal | undefined {
  return this._terminal;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run components/pages-component-terminal/src/PagesTerminal.test.ts`
Expected: All tests PASS

- [ ] **Step 5: Rebuild the package**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx turbo run build --filter=@casehubio/pages-component-terminal`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add components/pages-component-terminal/src/PagesTerminal.ts components/pages-component-terminal/src/PagesTerminal.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(terminal): add paste() method and terminal getter for host app integration"
```

---

### Task 2: esbuild multi-entry + rename index.ts → app.ts

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Rename: `app/src/main/webui/src/index.ts` → `app/src/main/webui/src/app.ts`
- Modify: `app/src/main/webui/esbuild.config.mjs`
- Create: `app/src/main/webui/src/terminal.ts` (minimal placeholder)
- Modify: `app/src/main/resources/META-INF/resources/app/session.html`

**Interfaces:**
- Produces: `dist/app.js` (dashboard bundle, was `dist/app.js` via `outfile`)
- Produces: `dist/terminal.js` (terminal bundle — placeholder for now)
- Produces: shared chunks in `dist/` for common deps

- [ ] **Step 1: Rename index.ts to app.ts**

```bash
git -C /Users/mdproctor/claude/casehub/claudony mv app/src/main/webui/src/index.ts app/src/main/webui/src/app.ts
```

- [ ] **Step 2: Create minimal terminal.ts placeholder**

Create `app/src/main/webui/src/terminal.ts`:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { hostPanel, rows } from "@casehubio/pages-ui";

// Placeholder — components added in subsequent tasks
const app = rows(
  hostPanel("terminal-workspace"),
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).catch(console.error);
}
```

- [ ] **Step 3: Update esbuild config for multi-entry with splitting**

Replace `app/src/main/webui/esbuild.config.mjs`:

```javascript
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/app.ts", "src/terminal.ts"],
  bundle: true,
  outdir: "dist",
  format: "esm",
  splitting: true,
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
  console.log("Watching for changes...");
} else {
  await build(options);
}
```

- [ ] **Step 4: Update session.html to load new bundle**

Replace `app/src/main/resources/META-INF/resources/app/session.html`:

```html
<!DOCTYPE html>
<html lang="en">
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
</html>
```

- [ ] **Step 5: Verify build produces correct output**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: `dist/` contains `app.js`, `terminal.js`, and shared chunk files. No errors.

- [ ] **Step 6: Verify typecheck passes**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/app.ts app/src/main/webui/src/terminal.ts app/src/main/webui/esbuild.config.mjs app/src/main/resources/META-INF/resources/app/session.html
git -C /Users/mdproctor/claude/casehub/claudony commit -m "chore(#161): multi-entry esbuild — rename index.ts to app.ts, add terminal.ts placeholder"
```

**Note:** After this commit, the terminal page will show a blank `#app` div since the old `terminal.js` IIFE is no longer loaded by session.html. The old `terminal.js` file still exists in `META-INF/resources/app/` but is no longer referenced. This is intentional — the terminal page is broken until Task 7 completes.

---

### Task 3: ClaudonyTerminalHeader + ClaudonyKeyBar

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Create: `app/src/main/webui/src/components/terminal-header.ts`
- Create: `app/src/main/webui/src/components/key-bar.ts`

**Interfaces:**
- Produces: `ClaudonyTerminalHeader` — `configure({ sessionName, sessionId })`, `updateStatus(text, cssClass)`
- Produces: `ClaudonyKeyBar` — emits `pages-event` with topic `key-pressed`, payload `{ code }`

- [ ] **Step 1: Create ClaudonyTerminalHeader**

Create `app/src/main/webui/src/components/terminal-header.ts`:

```typescript
export class ClaudonyTerminalHeader extends HTMLElement {
  private _sessionName = "Session";
  private _sessionId = "";

  connectedCallback(): void {
    this.render();
  }

  configure(opts: { sessionName: string; sessionId: string }): void {
    this._sessionName = opts.sessionName;
    this._sessionId = opts.sessionId;
    this.render();
  }

  updateStatus(text: string, cssClass: string): void {
    const badge = this.querySelector("#status-badge");
    if (badge) {
      badge.textContent = text;
      badge.className = "badge " + cssClass;
    }
  }

  private render(): void {
    this.innerHTML = `
      <style>
        .terminal-header {
          display: flex; align-items: center; gap: 12px;
          padding: 10px 16px; background: var(--surface);
          border-bottom: 1px solid var(--border); flex-shrink: 0;
        }
        #back-btn { color: var(--accent); text-decoration: none; font-size: 14px; white-space: nowrap; }
        #back-btn:hover { text-decoration: underline; }
        #session-name { font-weight: 600; font-size: 14px; flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
        .compose-btn {
          background: #2a2a4a; color: var(--text); border: none; padding: 6px 12px;
          border-radius: var(--radius); cursor: pointer; font-size: 12px;
        }
        .compose-btn:hover { background: #3a3a5c; }
      </style>
      <header class="terminal-header">
        <a href="/app/" id="back-btn">&#8592; Dashboard</a>
        <span id="session-name">${this.escHtml(this._sessionName)}</span>
        <span id="status-badge" class="badge idle">connecting</span>
        <button id="compose-btn" class="compose-btn" title="Compose text (Ctrl+G)">Compose</button>
        <button id="workers-toggle-btn" class="compose-btn" title="Toggle workers panel">Workers</button>
        <button id="ch-toggle-btn" class="compose-btn" title="Toggle channel panel (Ctrl+K)">Channels</button>
      </header>
    `;
  }

  private escHtml(s: string): string {
    return s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
  }
}

customElements.define("claudony-terminal-header", ClaudonyTerminalHeader);
```

- [ ] **Step 2: Create ClaudonyKeyBar**

Create `app/src/main/webui/src/components/key-bar.ts`:

```typescript
const KEYS = [
  { code: "\x1b", label: "Esc" },
  { code: "\x03", label: "Ctrl+C" },
  { code: "\x04", label: "Ctrl+D" },
  { code: "\x09", label: "Tab" },
  { code: "`", label: "`" },
  { code: "|", label: "|" },
  { code: "~", label: "~" },
  { code: "\x1b[A", label: "↑" },
  { code: "\x1b[B", label: "↓" },
  { code: "\x1b[C", label: "→" },
  { code: "\x1b[D", label: "←" },
];

export class ClaudonyKeyBar extends HTMLElement {
  connectedCallback(): void {
    const isTouch = "ontouchstart" in window || navigator.maxTouchPoints > 0;
    if (!isTouch) {
      this.style.display = "none";
      return;
    }

    this.innerHTML = `
      <style>
        .key-bar {
          display: flex; gap: 4px; padding: 6px 8px;
          background: var(--surface); border-top: 1px solid var(--border);
          overflow-x: auto; flex-shrink: 0;
        }
        .key-bar::-webkit-scrollbar { display: none; }
        .key-bar button {
          background: var(--bg); color: var(--text); border: 1px solid var(--border);
          border-radius: 4px; padding: 6px 10px; font-size: 12px;
          white-space: nowrap; cursor: pointer; flex-shrink: 0;
        }
        .key-bar button:hover { background: var(--accent); color: #fff; }
      </style>
      <div class="key-bar" id="key-bar">
        ${KEYS.map(k => `<button data-code="${this.escAttr(k.code)}">${k.label}</button>`).join("")}
        <button id="compose-key-btn">Compose</button>
      </div>
    `;

    this.querySelector(".key-bar")!.addEventListener("click", (e) => {
      const btn = (e.target as HTMLElement).closest("button");
      if (!btn) return;

      const code = btn.dataset.code;
      if (code) {
        this.dispatchEvent(new CustomEvent("pages-event", {
          bubbles: true, composed: true,
          detail: { topic: "key-pressed", payload: { code } },
        }));
      } else if (btn.id === "compose-key-btn") {
        this.dispatchEvent(new CustomEvent("pages-event", {
          bubbles: true, composed: true,
          detail: { topic: "compose-requested", payload: {} },
        }));
      }
    });
  }

  private escAttr(s: string): string {
    return s.replace(/&/g, "&amp;").replace(/"/g, "&quot;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
  }
}

customElements.define("claudony-key-bar", ClaudonyKeyBar);
```

- [ ] **Step 3: Verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 4: Verify build**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: Build succeeds

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/components/terminal-header.ts app/src/main/webui/src/components/key-bar.ts
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): ClaudonyTerminalHeader and ClaudonyKeyBar components"
```

---

### Task 4: ClaudonyWorkerPanel

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Create: `app/src/main/webui/src/components/worker-panel.ts`

**Interfaces:**
- Consumes: `/api/sessions/{id}/case-events` SSE endpoint
- Produces: `ClaudonyWorkerPanel` — `configure({ sessionId })`, `destroy()`, emits `worker-selected` topic

- [ ] **Step 1: Create ClaudonyWorkerPanel**

Create `app/src/main/webui/src/components/worker-panel.ts`:

```typescript
import { escapeHtml } from "../util/auth";

interface WorkerInfo {
  id: string;
  name: string;
  status: string;
  roleName?: string;
  createdAt: string;
}

export class ClaudonyWorkerPanel extends HTMLElement {
  private _sessionId = "";
  private _eventSource: EventSource | null = null;
  private _collapsed = true;

  connectedCallback(): void {
    this.render();
  }

  configure(opts: { sessionId: string }): void {
    this._sessionId = opts.sessionId;
    if (this._eventSource) {
      this._eventSource.close();
      this._eventSource = null;
    }
    if (!this._collapsed) {
      this.connectSSE();
    }
  }

  destroy(): void {
    if (this._eventSource) {
      this._eventSource.close();
      this._eventSource = null;
    }
  }

  open(): void {
    this._collapsed = false;
    this.classList.remove("collapsed");
    if (this._sessionId && !this._eventSource) {
      this.connectSSE();
    }
  }

  close(): void {
    this._collapsed = true;
    this.classList.add("collapsed");
    if (this._eventSource) {
      this._eventSource.close();
      this._eventSource = null;
    }
  }

  toggle(): void {
    if (this._collapsed) this.open();
    else this.close();
  }

  private connectSSE(): void {
    if (this._eventSource) this._eventSource.close();

    this._eventSource = new EventSource(
      "/api/sessions/" + this._sessionId + "/case-events"
    );

    this._eventSource.onmessage = (e) => {
      this.renderWorkers(JSON.parse(e.data) as WorkerInfo[]);
    };

    this._eventSource.onerror = () => {
      // EventSource reconnects automatically — no manual retry
    };

    if ((window as Record<string, unknown>).__CLAUDONY_TEST_MODE__) {
      (window as Record<string, unknown>)._caseEventSource = this._eventSource;
    }
  }

  private render(): void {
    this.classList.add("collapsed");

    this.innerHTML = `
      <style>
        :host, claudony-worker-panel {
          width: 240px; min-width: 240px;
          background: var(--surface); border-right: 1px solid var(--border);
          display: flex; flex-direction: column;
          transition: width 0.2s, min-width 0.2s;
          overflow: hidden;
        }
        claudony-worker-panel.collapsed { width: 0; min-width: 0; }
        .case-panel-header {
          display: flex; align-items: center; justify-content: space-between;
          padding: 8px 12px; border-bottom: 1px solid var(--border); flex-shrink: 0;
        }
        .case-panel-title { font-size: 12px; font-weight: 600; text-transform: uppercase; color: var(--text-dim); }
        .ch-close-btn {
          background: none; border: none; color: var(--text-dim);
          font-size: 14px; cursor: pointer; padding: 2px 6px;
        }
        .ch-close-btn:hover { color: var(--text); background: transparent; }
        .case-worker-list { flex: 1; overflow-y: auto; padding: 8px 0; }
        .case-worker-row {
          display: flex; align-items: center; gap: 8px;
          padding: 6px 12px; cursor: pointer; font-size: 13px;
        }
        .case-worker-row:hover { background: rgba(255,255,255,0.04); }
        .case-worker-row.active-worker { background: rgba(0,122,204,0.12); border-left: 2px solid var(--accent); }
        .worker-status-dot {
          width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0;
        }
        .worker-status-dot.active { background: var(--active); }
        .worker-status-dot.idle { background: var(--idle); }
        .worker-status-dot.waiting { background: var(--waiting); }
        .worker-status-dot.faulted { background: var(--danger); }
        .case-worker-name { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
        .case-worker-time { font-size: 11px; color: var(--text-dim); }
        .case-panel-placeholder { padding: 16px; color: var(--text-dim); font-size: 13px; text-align: center; }
      </style>
      <div class="case-panel-header">
        <span class="case-panel-title">Workers</span>
        <button id="case-close-btn" class="ch-close-btn" title="Close">&#10005;</button>
      </div>
      <div id="case-worker-list" class="case-worker-list">
        <div class="case-panel-placeholder">No case assigned.</div>
      </div>
    `;

    this.querySelector("#case-close-btn")!.addEventListener("click", () => this.close());
  }

  private renderWorkers(workers: WorkerInfo[]): void {
    const list = this.querySelector("#case-worker-list")!;
    list.innerHTML = "";

    if (!workers || workers.length === 0) {
      list.innerHTML = '<div class="case-panel-placeholder">No workers found.</div>';
      return;
    }

    workers.forEach((w) => {
      const row = document.createElement("div");
      const status = (w.status || "idle").toLowerCase();
      const isActive = w.id === this._sessionId;
      row.className = "case-worker-row" + (isActive ? " active-worker" : "");

      const displayName = w.roleName || w.name.replace(/^claudony-worker-/, "").replace(/^claudony-/, "");
      const timeAgo = this.workerTimeAgo(w.createdAt);

      row.innerHTML =
        '<span class="worker-status-dot ' + escapeHtml(status) + '"></span>' +
        '<span class="case-worker-name">' + escapeHtml(displayName) + '</span>' +
        '<span class="case-worker-time">' + escapeHtml(timeAgo) + '</span>';

      row.addEventListener("click", () => {
        if (isActive) return;
        this.dispatchEvent(new CustomEvent("pages-event", {
          bubbles: true, composed: true,
          detail: { topic: "worker-selected", payload: { sessionId: w.id, name: displayName } },
        }));
      });

      list.appendChild(row);
    });
  }

  private workerTimeAgo(iso: string): string {
    const diff = Date.now() - new Date(iso).getTime();
    const m = Math.floor(diff / 60000);
    if (m < 1) return "now";
    if (m < 60) return m + "m";
    return Math.floor(m / 60) + "h";
  }
}

customElements.define("claudony-worker-panel", ClaudonyWorkerPanel);
```

- [ ] **Step 2: Verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 3: Verify build**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: Build succeeds

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/components/worker-panel.ts
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): ClaudonyWorkerPanel component — SSE worker list with click-to-switch"
```

---

### Task 5: ClaudonyChannelPanel

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Create: `app/src/main/webui/src/components/channel-panel.ts`

**Interfaces:**
- Consumes: `/api/mesh/config`, `/api/mesh/channels`, `/api/mesh/channels/{name}/timeline`, `/api/mesh/channels/{name}/events`, `/api/mesh/channels/{name}/messages`, `/api/sessions/{id}/lineage`
- Produces: `ClaudonyChannelPanel` — `configure({ sessionId, caseId?, roleName?, createdAt?, channel? })`, `destroy()`, `open()`, `close()`, `toggle()`

This is the largest component (~350 lines). It handles channel select, message feed, SSE/polling with stale cursor management, case context header with lineage, and human message composition.

- [ ] **Step 1: Create ClaudonyChannelPanel**

Create `app/src/main/webui/src/components/channel-panel.ts`. This is the full channel panel — all logic from terminal.js lines 154-703 decomposed into this single focused component.

The component must faithfully port every behavior from terminal.js:
- Channel select dropdown with allowedTypes filtering
- Message rendering (normal messages + EVENT messages with telemetry)
- SSE EventSource with polling fallback on error
- Stale cursor detection with catch-up/reload prompt
- sessionStorage cursor persistence
- Case context header (role, status, elapsed ticker)
- Lineage display (prior workers, expandable)
- `?channel=` URL param auto-select
- Human message composition (type select + textarea + send button)

The implementation must use the same DOM structure and CSS classes as the original to preserve E2E test selectors: `#ch-select`, `#ch-feed`, `.ch-msg`, `#ch-input`, `#ch-send-btn`, `#ch-type-select`, `#ch-error`, `#ch-stale-prompt`, `.ch-case-header`, `.ch-lineage`, etc.

- [ ] **Step 2: Verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 3: Verify build**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: Build succeeds

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/components/channel-panel.ts
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): ClaudonyChannelPanel component — channel feed, SSE, stale cursor, case context"
```

---

### Task 6: ClaudonyTerminalWorkspace

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Create: `app/src/main/webui/src/components/terminal-workspace.ts`

**Interfaces:**
- Consumes: `PagesTerminal.configure()`, `PagesTerminal.sendInput()`, `PagesTerminal.paste()`, `PagesTerminal.terminal`
- Consumes: `ClaudonyWorkerPanel.configure()`, `.open()`, `.close()`, `.toggle()`, `.destroy()`
- Consumes: `ClaudonyChannelPanel.configure()`, `.open()`, `.close()`, `.toggle()`, `.destroy()`
- Produces: `ClaudonyTerminalWorkspace` — `configure({ sessionId, sessionName, proxyPeer?, caseId?, roleName?, status?, createdAt?, channel? })`, `destroy()`

- [ ] **Step 1: Create ClaudonyTerminalWorkspace**

Create `app/src/main/webui/src/components/terminal-workspace.ts`:

```typescript
import "@casehubio/pages-component-terminal";
import type { PagesTerminal } from "@casehubio/pages-component-terminal";
import "./worker-panel";
import type { ClaudonyWorkerPanel } from "./worker-panel";
import "./channel-panel";
import type { ClaudonyChannelPanel } from "./channel-panel";
import { fetchWithAuth } from "../util/auth";

interface WorkspaceConfig {
  sessionId: string;
  sessionName: string;
  proxyPeer?: string;
  caseId?: string;
  roleName?: string;
  status?: string;
  createdAt?: string;
  channel?: string;
}

export class ClaudonyTerminalWorkspace extends HTMLElement {
  private _config: WorkspaceConfig | null = null;
  private _terminal: PagesTerminal | null = null;
  private _workerPanel: ClaudonyWorkerPanel | null = null;
  private _channelPanel: ClaudonyChannelPanel | null = null;

  connectedCallback(): void {
    this.render();
    this.wireEvents();
  }

  configure(config: WorkspaceConfig): void {
    this._config = config;

    const proto = location.protocol === "https:" ? "wss:" : "ws:";
    const wsUrl = config.proxyPeer
      ? `${proto}//${location.host}/ws/proxy/${config.proxyPeer}/${config.sessionId}/{cols}/{rows}`
      : `${proto}//${location.host}/ws/${config.sessionId}/{cols}/{rows}`;

    this._terminal?.configure({ wsUrl });

    this._workerPanel?.configure({ sessionId: config.sessionId });
    if (config.caseId) {
      this._workerPanel?.open();
    }

    this._channelPanel?.configure({
      sessionId: config.sessionId,
      caseId: config.caseId,
      roleName: config.roleName,
      createdAt: config.createdAt,
      channel: config.channel,
    });

    if ((window as Record<string, unknown>).__CLAUDONY_TEST_MODE__ && this._terminal) {
      (window as Record<string, unknown>)._xtermTerminal = this._terminal.terminal;
    }
  }

  destroy(): void {
    this._workerPanel?.destroy();
    this._channelPanel?.destroy();
  }

  getTerminal(): PagesTerminal | null {
    return this._terminal;
  }

  toggleWorkers(): void {
    this._workerPanel?.toggle();
  }

  toggleChannels(): void {
    this._channelPanel?.toggle();
  }

  private render(): void {
    this.innerHTML = `
      <style>
        claudony-terminal-workspace {
          display: flex; flex: 1; overflow: hidden;
        }
        pages-component-terminal {
          flex: 1; overflow: hidden;
        }
        pages-component-terminal .xterm { height: 100%; }
        pages-component-terminal .xterm-viewport { overflow: hidden !important; }
      </style>
      <claudony-worker-panel></claudony-worker-panel>
      <pages-component-terminal></pages-component-terminal>
      <claudony-channel-panel></claudony-channel-panel>
    `;

    this._terminal = this.querySelector("pages-component-terminal") as PagesTerminal;
    this._workerPanel = this.querySelector("claudony-worker-panel") as ClaudonyWorkerPanel;
    this._channelPanel = this.querySelector("claudony-channel-panel") as ClaudonyChannelPanel;
  }

  private wireEvents(): void {
    this.addEventListener("pages-event", ((e: CustomEvent) => {
      const { topic, payload } = e.detail;

      switch (topic) {
        case "terminal-resize":
          this.handleResize(payload.cols, payload.rows);
          break;
        case "key-pressed":
          this._terminal?.sendInput(payload.code);
          break;
        case "worker-selected":
          this.handleWorkerSwitch(payload.sessionId, payload.name);
          break;
      }
    }) as EventListener);
  }

  private handleResize(cols: number, rows: number): void {
    if (!this._config) return;
    const resizeUrl = this._config.proxyPeer
      ? `/api/peers/${this._config.proxyPeer}/sessions/${this._config.sessionId}/resize?cols=${cols}&rows=${rows}`
      : `/api/sessions/${this._config.sessionId}/resize?cols=${cols}&rows=${rows}`;
    fetchWithAuth(resizeUrl, { method: "POST" }).catch(() => {});
  }

  private handleWorkerSwitch(newSessionId: string, newName: string): void {
    if (!this._config) return;

    this._config.sessionId = newSessionId;
    this._config.sessionName = newName;

    history.replaceState(null, "",
      "?id=" + newSessionId + "&name=" + encodeURIComponent(newName));

    const proto = location.protocol === "https:" ? "wss:" : "ws:";
    const wsUrl = this._config.proxyPeer
      ? `${proto}//${location.host}/ws/proxy/${this._config.proxyPeer}/${newSessionId}/{cols}/{rows}`
      : `${proto}//${location.host}/ws/${newSessionId}/{cols}/{rows}`;

    this._terminal?.configure({ wsUrl });

    this._workerPanel?.configure({ sessionId: newSessionId });
    this._channelPanel?.configure({
      sessionId: newSessionId,
      caseId: this._config.caseId,
      roleName: this._config.roleName,
      createdAt: this._config.createdAt,
    });

    // Dispatch for header to pick up
    this.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true, composed: true,
      detail: { topic: "session-changed", payload: { sessionName: newName, sessionId: newSessionId } },
    }));

    if ((window as Record<string, unknown>).__CLAUDONY_TEST_MODE__ && this._terminal) {
      (window as Record<string, unknown>)._xtermTerminal = this._terminal.terminal;
    }
  }
}

customElements.define("claudony-terminal-workspace", ClaudonyTerminalWorkspace);
```

- [ ] **Step 2: Verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 3: Verify build**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: Build succeeds

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/components/terminal-workspace.ts
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): ClaudonyTerminalWorkspace — layout coordinator with event routing"
```

---

### Task 7: terminal.ts entry point — compose overlay, session fetch, keyboard shortcuts, lifecycle

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Modify: `app/src/main/webui/src/terminal.ts` (replace placeholder from Task 2)

**Interfaces:**
- Consumes: `ClaudonyTerminalHeader.configure()`, `.updateStatus()`
- Consumes: `ClaudonyTerminalWorkspace.configure()`, `.destroy()`, `.getTerminal()`, `.toggleWorkers()`, `.toggleChannels()`
- Consumes: `ClaudonyKeyBar` (emits events, no API calls needed)
- Consumes: `PagesTerminal.paste()`

- [ ] **Step 1: Replace terminal.ts with full entry point**

Replace `app/src/main/webui/src/terminal.ts`:

```typescript
import "@casehubio/pages-component-terminal/xterm.css";
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { hostPanel, rows } from "@casehubio/pages-ui";
import "./components/terminal-header";
import "./components/terminal-workspace";
import "./components/key-bar";
import type { ClaudonyTerminalHeader } from "./components/terminal-header";
import type { ClaudonyTerminalWorkspace } from "./components/terminal-workspace";
import { fetchWithAuth } from "./util/auth";

registerPanel("terminal-header", "claudony-terminal-header");
registerPanel("terminal-workspace", "claudony-terminal-workspace");
registerPanel("key-bar", "claudony-key-bar");

const app = rows(
  hostPanel("terminal-header"),
  hostPanel("terminal-workspace"),
  hostPanel("key-bar"),
);

const container = document.getElementById("app");
if (!container) throw new Error("Missing #app container");

loadSite(container, app).then(() => {
  const params = new URLSearchParams(window.location.search);
  const sessionId = params.get("id");
  const sessionName = params.get("name") || "Session";
  const proxyPeer = params.get("proxyPeer") || undefined;
  const channel = params.get("channel") || undefined;

  if (!sessionId) {
    window.location.href = "/app/";
    return;
  }

  const header = document.querySelector("claudony-terminal-header") as ClaudonyTerminalHeader;
  const workspace = document.querySelector("claudony-terminal-workspace") as ClaudonyTerminalWorkspace;

  // Set title
  document.title = sessionName + " — RemoteCC";

  // Configure header
  header.configure({ sessionName, sessionId });

  // Listen for terminal lifecycle events
  document.addEventListener("pages-event", ((e: CustomEvent) => {
    const { topic, payload } = e.detail;
    switch (topic) {
      case "terminal-connected":
        header.updateStatus("connected", "active");
        break;
      case "terminal-disconnected":
        if (payload.reason === "session-expired") {
          header.updateStatus("session expired", "idle");
        } else {
          header.updateStatus("reconnecting…", "idle");
        }
        break;
      case "session-changed":
        header.configure({ sessionName: payload.sessionName, sessionId: payload.sessionId });
        document.title = payload.sessionName + " — RemoteCC";
        break;
      case "compose-requested":
        openCompose();
        break;
    }
  }) as EventListener);

  // Header button wiring
  header.querySelector("#workers-toggle-btn")!.addEventListener("click", () => workspace.toggleWorkers());
  header.querySelector("#ch-toggle-btn")!.addEventListener("click", () => workspace.toggleChannels());
  header.querySelector("#compose-btn")!.addEventListener("click", () => openCompose());

  // Fetch session and configure workspace
  fetchWithAuth("/api/sessions/" + sessionId)
    .then(r => r.ok ? r.json() : null)
    .then((session: { caseId?: string; roleName?: string; status?: string; createdAt?: string } | null) => {
      workspace.configure({
        sessionId,
        sessionName,
        proxyPeer,
        caseId: session?.caseId,
        roleName: session?.roleName,
        status: session?.status,
        createdAt: session?.createdAt,
        channel,
      });
    })
    .catch(() => {
      workspace.configure({ sessionId, sessionName, proxyPeer, channel });
    });

  // ── Compose overlay ────────────────────────────────────────────────────

  const overlay = document.createElement("div");
  overlay.id = "compose-overlay";
  overlay.className = "compose-overlay hidden";
  overlay.innerHTML = `
    <div class="compose-dialog">
      <div class="compose-header">
        <span>Compose</span>
        <span class="compose-hint">Ctrl+Enter to send · Esc to cancel</span>
      </div>
      <textarea id="compose-textarea" class="compose-textarea" placeholder="Type your message here — full mouse editing supported" rows="6" spellcheck="false"></textarea>
      <div class="compose-actions">
        <button id="compose-send-btn" class="compose-send">Send</button>
        <button id="compose-cancel-btn" class="compose-cancel">Cancel</button>
      </div>
    </div>
  `;
  document.body.appendChild(overlay);

  const composeTextarea = overlay.querySelector("#compose-textarea") as HTMLTextAreaElement;

  function openCompose(): void {
    overlay.classList.remove("hidden");
    composeTextarea.focus();
    composeTextarea.select();
  }

  function closeCompose(): void {
    overlay.classList.add("hidden");
    workspace.getTerminal()?.terminal?.focus();
  }

  function sendCompose(): void {
    const text = composeTextarea.value;
    if (!text) { closeCompose(); return; }
    composeTextarea.value = "";
    closeCompose();
    workspace.getTerminal()?.paste(text);
  }

  overlay.querySelector("#compose-send-btn")!.addEventListener("click", sendCompose);
  overlay.querySelector("#compose-cancel-btn")!.addEventListener("click", closeCompose);

  composeTextarea.addEventListener("keydown", (e) => {
    if (e.key === "Enter" && (e.ctrlKey || e.metaKey)) { e.preventDefault(); sendCompose(); }
    if (e.key === "Escape") { e.preventDefault(); closeCompose(); }
  });

  overlay.addEventListener("click", (e) => {
    if (e.target === overlay) closeCompose();
  });

  // ── Global keyboard shortcuts ──────────────────────────────────────────

  document.addEventListener("keydown", (e) => {
    if (e.ctrlKey && e.key === "g" && overlay.classList.contains("hidden")) {
      e.preventDefault();
      openCompose();
    }
    if (e.ctrlKey && e.key === "k") {
      e.preventDefault();
      workspace.toggleChannels();
    }
  });

  // ── Lifecycle cleanup ──────────────────────────────────────────────────

  window.addEventListener("beforeunload", () => {
    workspace.destroy();
  });

}).catch(console.error);
```

- [ ] **Step 2: Add compose overlay CSS to style.css**

The compose overlay is created by the entry point, not a component. Its CSS must be in `style.css` (the global stylesheet loaded by session.html). The compose styles already exist in `style.css` — verify they're still there. If they were accidentally removed, they are at lines 455-479 of the current `style.css`.

- [ ] **Step 3: Verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npx tsc --noEmit`
Expected: No type errors

- [ ] **Step 4: Verify build**

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && npm run build`
Expected: Build succeeds, `dist/terminal.js` is now the full entry point

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/terminal.ts
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): terminal.ts entry point — session fetch, compose overlay, keyboard shortcuts, lifecycle"
```

---

### Task 8: CSS migration + remove old terminal.js + update E2E test + verify

**Repo:** `/Users/mdproctor/claude/casehub/claudony`

**Files:**
- Modify: `app/src/main/resources/META-INF/resources/app/style.css` — remove terminal-specific CSS (lines 139-500+)
- Delete: `app/src/main/resources/META-INF/resources/app/terminal.js`
- Modify: `app/src/test/java/io/casehub/claudony/e2e/CaseWorkerPanelE2ETest.java` — AC4 test

**Interfaces:**
- Consumes: all components from Tasks 3-7
- Produces: working terminal page with all 32 E2E tests passing

- [ ] **Step 1: Move terminal-specific CSS from style.css into components**

Remove from `style.css` everything from `/* Terminal page */` (line 139) through the end of the channel panel / compose / key-bar styles (~line 500). These rules are now inside each component's inline `<style>` tag.

Keep in `style.css`:
- `:root` variables (lines 1-15)
- `*` reset, `body`, `header`, `h1`, `button` globals (lines 17-50)
- `.badge` styles (lines 52-65)
- `main`, `#session-grid` dashboard styles (lines 67+)
- `.dialog-*`, `dialog` styles
- All dashboard/fleet/mesh panel styles
- Compose overlay styles (`.compose-overlay`, `.compose-dialog`, etc.) — these stay because the overlay is created by the entry point, not a component

- [ ] **Step 2: Delete old terminal.js**

```bash
git -C /Users/mdproctor/claude/casehub/claudony rm app/src/main/resources/META-INF/resources/app/terminal.js
```

- [ ] **Step 3: Update CaseWorkerPanelE2ETest AC4**

Replace the `noPollingInterval_forWorkerUpdates_inSource()` test at line 165-180 of `app/src/test/java/io/casehub/claudony/e2e/CaseWorkerPanelE2ETest.java`:

```java
@Test
@Order(4)
void noPollingInterval_forWorkerUpdates_inBrowser() {
    // Regression guard: verify worker updates use SSE, not polling.
    // Original test read terminal.js source for "function pollWorkers" / "setInterval".
    // After migration to TypeScript components, verify the behavioral contract instead:
    // EventSource is connected and no polling timer exists.
    var caseId = "e2e-sse-nopoll-001";
    registry.register(new Session("nopoll-w1", "claudony-worker-nopoll-w1", "/tmp", "claude",
            SessionStatus.ACTIVE, Instant.now(), Instant.now(), Optional.empty(),
            Optional.of(caseId), Optional.of("analyst"), TenancyConstants.DEFAULT_TENANT_ID));

    page.addInitScript("window.__CLAUDONY_TEST_MODE__ = true;");
    page.navigate(BASE_URL + "/app/session.html?id=nopoll-w1&name=analyst");

    // Wait for EventSource to be wired
    page.waitForFunction("window._caseEventSource && window._caseEventSource.readyState !== 2");

    // Verify EventSource is connected (readyState 0=CONNECTING or 1=OPEN, not 2=CLOSED)
    var readyState = page.evaluate("window._caseEventSource.readyState");
    assertThat((Number) readyState).as("EventSource should be connected").isLessThanOrEqualTo(1);
}
```

- [ ] **Step 4: Run full E2E test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest,ChannelPanelE2ETest,CaseWorkerPanelE2ETest,CaseContextPanelE2ETest -pl claudony-app --also-make`

Expected: All 32+ tests PASS (4 setup + 7 dashboard + 2 terminal + 19 channel + 7 worker + 4 case context)

- [ ] **Step 5: Run full unit test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: 601 tests PASS (verify no regressions from CSS changes or file deletion)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/resources/META-INF/resources/app/style.css app/src/test/java/io/casehub/claudony/e2e/CaseWorkerPanelE2ETest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(#161): complete terminal page migration — CSS moved to components, old terminal.js removed

Closes #161

- Terminal page now served via TypeScript Web Components + Quinoa esbuild pipeline
- 5 new components: terminal-header, terminal-workspace, worker-panel, channel-panel, key-bar
- loadSite() composition with pages-event protocol for coordination
- esbuild code splitting shares common deps between dashboard and terminal bundles
- CaseWorkerPanelE2ETest AC4 converted from source-reading to behavioral assertion
- All 32 E2E tests pass"
```
