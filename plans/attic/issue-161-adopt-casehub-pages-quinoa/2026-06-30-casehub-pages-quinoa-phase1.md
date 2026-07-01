# casehub-pages Quinoa Adoption — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire Quinoa + esbuild into Claudony and render the session grid via `loadSite()` with a `claudony-session-grid` Web Component — proving the casehub-pages integration end-to-end.

**Architecture:** Quinoa builds TypeScript in `app/src/main/webui/` into `dist/`, served at `/app/` via `quarkus.quinoa.ui-root-path=/app`. The entry point calls `loadSite()` with a minimal layout tree containing a single `hostPanel("session-grid")`. The `claudony-session-grid` Web Component replicates the current session grid's rendering and REST polling internally. Old static files coexist under `/app/` — `session.html` and `terminal.js` remain functional. Session clicks navigate to the old `/app/session.html?id=...` page (one line of throwaway code replaced in phase 3).

**Tech Stack:** Quarkus 3.32.2, quarkus-quinoa 2.8.3, esbuild, TypeScript, `@casehubio/pages-runtime` 0.2.0, `@casehubio/pages-ui` 0.2.0

## Global Constraints

- Java 21 API surface compiled on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Use `mvn` not `./mvnw`
- All tests run with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Quarkus test port is random (`quarkus.http.test-port=0`) — never hardcode port 8081
- `@TestSecurity(user = "test", roles = "user")` only on classes that exercise HTTP endpoints
- Commit messages reference `#161`: `feat(#161): ...` or `Refs #161`
- Auth protection: `quarkus.http.auth.permission.protected.paths=/api/*,/ws/*,/app/*` — Quinoa output must be served under `/app/` to stay covered
- Garden context: GE-20260629-59c7e6 — avoid interpolating TypeScript constants inside `html()` template literals containing `<script>` blocks (esbuild minifies them)
- Package test command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --also-make`

---

### Task 1: Add Quinoa Extension and Configure

**Files:**
- Modify: `app/pom.xml` — add `quarkus-quinoa` dependency
- Modify: `app/src/main/resources/application.properties` — add Quinoa config
- Create: `app/src/main/webui/package.json`
- Create: `app/src/main/webui/esbuild.config.mjs`
- Create: `app/src/main/webui/tsconfig.json`
- Create: `app/src/main/webui/.npmrc`
- Create: `app/src/main/webui/public/index.html`
- Create: `app/src/main/webui/src/index.ts`
- Test: `app/src/test/java/io/casehub/claudony/frontend/StaticFilesTest.java` (modify)
- Test: `app/src/test/java/io/casehub/claudony/frontend/AppAuthProtectionTest.java` (modify)

**Interfaces:**
- Consumes: nothing (first task)
- Produces: Quinoa build pipeline producing `dist/app.js` served at `/app/app.js`. `loadSite()` renders a placeholder `<h1>Claudony</h1>` into `<div id="app">`. Old files (`session.html`, `terminal.js`, `dashboard.js`, `style.css`) still accessible at `/app/session.html` etc.

- [ ] **Step 1: Write failing test — Quinoa-served app.js is accessible**

Add a test that verifies `/app/app.js` is served with JavaScript content type. This will fail because Quinoa isn't wired yet.

```java
// In StaticFilesTest.java, add:
@Test
void quinoaBundleIsAccessible() {
    given().when().get("/app/app.js")
        .then().statusCode(200)
        .contentType(containsString("javascript"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --also-make -Dtest=StaticFilesTest#quinoaBundleIsAccessible`
Expected: FAIL — no `/app/app.js` exists yet.

- [ ] **Step 3: Add quarkus-quinoa dependency to app/pom.xml**

Add inside `<dependencies>`:

```xml
<!-- Quinoa — builds src/main/webui TypeScript and serves the output -->
<dependency>
  <groupId>io.quarkiverse.quinoa</groupId>
  <artifactId>quarkus-quinoa</artifactId>
  <version>2.8.3</version>
</dependency>
```

- [ ] **Step 4: Add Quinoa configuration to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# Quinoa — TypeScript frontend build via esbuild
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
quarkus.quinoa.ui-root-path=/app
```

`ui-root-path=/app` serves Quinoa output under `/app/` which is already covered by the auth protection rule on line 69: `quarkus.http.auth.permission.protected.paths=/api/*,/ws/*,/app/*`.

- [ ] **Step 5: Create the webui directory from the quinoa-host template**

Create `app/src/main/webui/.npmrc`:
```
@casehubio:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

Create `app/src/main/webui/package.json`:
```json
{
  "name": "claudony-webui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "0.2.0",
    "@casehubio/pages-ui": "0.2.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

Create `app/src/main/webui/esbuild.config.mjs`:
```javascript
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
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

Create `app/src/main/webui/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

- [ ] **Step 6: Create the minimal entry point and HTML shell**

Create `app/src/main/webui/public/index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="theme-color" content="#007acc">
    <title>Claudony</title>
    <link rel="manifest" href="/manifest.json">
</head>
<body>
    <div id="app"></div>
    <script type="module" src="app.js"></script>
    <script>
    if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register('/sw.js');
    }
    </script>
</body>
</html>
```

Create `app/src/main/webui/src/index.ts`:
```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page } from "@casehubio/pages-ui";

const app = page("Claudony");

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).catch(console.error);
}
```

- [ ] **Step 7: Run the test to verify Quinoa serves the bundle**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --also-make -Dtest=StaticFilesTest#quinoaBundleIsAccessible`
Expected: PASS — Quinoa builds `dist/app.js` and serves it at `/app/app.js`.

- [ ] **Step 8: Update StaticFilesTest for coexistence — old files still accessible**

The old `index.html` at `META-INF/resources/app/index.html` is now **shadowed** by Quinoa's `index.html` at `/app/index.html`. Update the test that checks for `dashboard.js` in `index.html` — the new Quinoa `index.html` loads `app.js` instead:

```java
@Test
void indexHtmlLoadsAppBundle() {
    given().when().get("/app/index.html")
        .then().statusCode(200)
        .body(containsString("app.js"));
}
```

Remove or replace the old assertions that check for `dashboard.js` and `mesh-panel` in `index.html` (the Quinoa `index.html` replaces the old one). Keep the `session.html`, `terminal.js`, `style.css` tests — those old files are still served during coexistence.

Remove `dashboardContainsDashboardScript`, `dashboardContainsMeshPanel`, `indexHtml_containsMeshDock`.

Update `AppAuthProtectionTest` — keep the `index.html` test (still protected under `/app/*`), keep `session.html` test (still exists):

```java
@Test
void unauthenticatedDashboardIsRedirectedToLogin() {
    given().redirects().follow(false)
        .when().get("/app/index.html")
        .then().statusCode(allOf(greaterThanOrEqualTo(300), lessThan(400)));
}

@Test
void unauthenticatedSessionPageIsRedirectedToLogin() {
    given().redirects().follow(false)
        .when().get("/app/session.html")
        .then().statusCode(allOf(greaterThanOrEqualTo(300), lessThan(400)));
}
```

No change needed to `AppAuthProtectionTest` — both assertions still hold.

- [ ] **Step 9: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All 601+ tests pass. The old files coexist with Quinoa output.

- [ ] **Step 10: Commit**

```bash
git add app/pom.xml app/src/main/resources/application.properties app/src/main/webui/ app/src/test/java/io/casehub/claudony/frontend/StaticFilesTest.java
git commit -m "feat(#161): wire Quinoa + esbuild — loadSite() renders placeholder

Adds quarkus-quinoa 2.8.3 to app module. Quinoa builds
app/src/main/webui/src/index.ts → dist/app.js, served at /app/.
Old static files coexist during migration.

Refs #161"
```

---

### Task 2: Session Grid Web Component — Rendering and Polling

**Files:**
- Create: `app/src/main/webui/src/components/session-grid.ts` — `claudony-session-grid` Web Component
- Create: `app/src/main/webui/src/util/time.ts` — `timeAgo()` helper
- Create: `app/src/main/webui/src/util/auth.ts` — 401 handler
- Create: `app/src/main/webui/src/theme.ts` — CSS custom properties / design tokens
- Modify: `app/src/main/webui/src/index.ts` — register panel, add `hostPanel` to layout tree

**Interfaces:**
- Consumes: Task 1's Quinoa pipeline. `loadSite()`, `hostPanel()`, `registerPanel()` from `@casehubio/pages-ui` and `@casehubio/pages-runtime`.
- Produces: `claudony-session-grid` custom element that fetches `/api/sessions` every 5s, renders session cards with status badges, working directory, time-ago, and click-to-open navigation. Fires `pages-event` with `topic: "session-selected"` (not wired to `activateSlot` yet — phase 1 navigates to old `session.html`). Handles 401 responses with auth modal.

- [ ] **Step 1: Create the time utility**

Create `app/src/main/webui/src/util/time.ts`:

```typescript
export function timeAgo(iso: string): string {
  const diff = Date.now() - new Date(iso).getTime();
  const m = Math.floor(diff / 60000);
  if (m < 1) return "just now";
  if (m < 60) return `${m}m ago`;
  const h = Math.floor(m / 60);
  if (h < 24) return `${h}h ago`;
  return `${Math.floor(h / 24)}d ago`;
}
```

- [ ] **Step 2: Create the auth utility**

Create `app/src/main/webui/src/util/auth.ts`:

```typescript
let authModalShown = false;

export async function fetchWithAuth(url: string, init?: RequestInit): Promise<Response> {
  const res = await fetch(url, init);
  if (res.status === 401 && !authModalShown) {
    authModalShown = true;
    showAuthModal();
  }
  return res;
}

function showAuthModal(): void {
  const overlay = document.createElement("div");
  overlay.style.cssText =
    "position:fixed;inset:0;background:rgba(0,0,0,0.6);display:flex;align-items:center;justify-content:center;z-index:9999";

  const box = document.createElement("div");
  box.style.cssText =
    "background:#252526;padding:2rem;border-radius:8px;color:#ccc;text-align:center;max-width:320px";
  box.innerHTML = `
    <h3 style="margin:0 0 1rem">Session expired</h3>
    <a href="/auth/login" style="color:#007acc">Log in</a>
  `;

  overlay.appendChild(box);
  document.body.appendChild(overlay);
}
```

- [ ] **Step 3: Create the theme module**

Create `app/src/main/webui/src/theme.ts`:

```typescript
export const THEME_CSS = `
  :host {
    --bg: #1e1e1e;
    --surface: #252526;
    --border: #3e3e42;
    --text: #cccccc;
    --text-muted: #888;
    --accent: #007acc;
    --active: #4ec9b0;
    --danger: #f44747;
    --radius: 6px;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    color: var(--text);
  }
`;
```

- [ ] **Step 4: Create the session grid Web Component**

Create `app/src/main/webui/src/components/session-grid.ts`:

```typescript
import { timeAgo } from "../util/time";
import { fetchWithAuth } from "../util/auth";
import { THEME_CSS } from "../theme";

interface Session {
  id: string;
  name: string;
  status: string;
  workingDir: string;
  lastActive: string;
  caseId?: string;
  roleName?: string;
  instanceUrl?: string;
  instanceName?: string;
  stale?: boolean;
}

const POLL_INTERVAL = 5000;

export class ClaudonySessionGrid extends HTMLElement {
  private shadow: ShadowRoot;
  private sessions: Session[] = [];
  private pollTimer: number | null = null;

  constructor() {
    super();
    this.shadow = this.attachShadow({ mode: "open" });
  }

  connectedCallback(): void {
    this.render();
    this.fetchSessions();
    this.pollTimer = window.setInterval(() => this.fetchSessions(), POLL_INTERVAL);
  }

  disconnectedCallback(): void {
    if (this.pollTimer !== null) {
      clearInterval(this.pollTimer);
      this.pollTimer = null;
    }
  }

  private async fetchSessions(): Promise<void> {
    try {
      const res = await fetchWithAuth("/api/sessions");
      if (!res.ok) return;
      this.sessions = await res.json();
      this.renderGrid();
    } catch {
      // network error — keep showing stale data
    }
  }

  private render(): void {
    this.shadow.innerHTML = `
      <style>
        ${THEME_CSS}
        :host { display: block; height: 100%; overflow-y: auto; padding: 1rem; }
        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
        .header h2 { margin: 0; font-size: 1.1rem; }
        .new-btn {
          background: var(--accent); color: #fff; border: none; padding: 0.4rem 0.8rem;
          border-radius: var(--radius); cursor: pointer; font-size: 0.85rem;
        }
        .new-btn:hover { filter: brightness(1.2); }
        .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 0.75rem; }
        .card {
          background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius);
          padding: 0.75rem; cursor: pointer; transition: border-color 0.15s;
        }
        .card:hover { border-color: var(--accent); }
        .card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.4rem; }
        .card-name { font-weight: 600; font-size: 0.95rem; }
        .badge {
          font-size: 0.7rem; padding: 0.15rem 0.4rem; border-radius: 3px;
          text-transform: uppercase; font-weight: 600;
        }
        .badge-active { background: #2d4a2d; color: var(--active); }
        .badge-waiting { background: #4a3d2d; color: #e0a050; }
        .badge-idle { background: #3e3e42; color: var(--text-muted); }
        .card-dir { font-size: 0.8rem; color: var(--text-muted); margin-bottom: 0.3rem; font-family: monospace; }
        .card-time { font-size: 0.75rem; color: var(--text-muted); }
        .card-actions { display: flex; gap: 0.4rem; margin-top: 0.5rem; }
        .action-btn {
          font-size: 0.75rem; padding: 0.2rem 0.5rem; border: 1px solid var(--border);
          background: transparent; color: var(--text-muted); border-radius: 3px; cursor: pointer;
        }
        .action-btn:hover { border-color: var(--accent); color: var(--accent); }
        .action-btn.danger:hover { border-color: var(--danger); color: var(--danger); }
        .stale { opacity: 0.6; }
        .stale-badge { font-size: 0.7rem; color: #e0a050; margin-bottom: 0.3rem; }
        .instance-badge { font-size: 0.7rem; background: #333; padding: 0.1rem 0.3rem; border-radius: 3px; margin-left: 0.4rem; }
        .empty { text-align: center; color: var(--text-muted); padding: 3rem 1rem; }
      </style>
      <div class="header">
        <h2>Sessions</h2>
        <button class="new-btn" id="new-btn">+ New Session</button>
      </div>
      <div class="grid" id="grid"></div>
    `;

    this.shadow.getElementById("new-btn")!.addEventListener("click", () => this.showNewSessionDialog());
  }

  private renderGrid(): void {
    const grid = this.shadow.getElementById("grid")!;
    if (this.sessions.length === 0) {
      grid.innerHTML = '<div class="empty">No sessions running</div>';
      return;
    }

    grid.innerHTML = this.sessions.map((s) => {
      const name = s.name.replace(/^claudony-/, "");
      const status = s.status.toLowerCase();
      const badgeClass = status === "active" ? "badge-active" : status === "waiting" ? "badge-waiting" : "badge-idle";
      const instanceBadge = s.instanceUrl
        ? `<span class="instance-badge">${s.instanceName || s.instanceUrl}</span>`
        : "";
      const staleBadge = s.stale ? `<div class="stale-badge">⏰ last seen ${timeAgo(s.lastActive)}</div>` : "";

      return `
        <div class="card ${s.stale ? "stale" : ""}" data-id="${s.id}" data-name="${encodeURIComponent(name)}">
          <div class="card-header">
            <span class="card-name">${name}${instanceBadge}</span>
            <span class="badge ${badgeClass}">${s.status}</span>
          </div>
          ${staleBadge}
          <div class="card-dir">${s.workingDir || ""}</div>
          <div class="card-time">Active ${timeAgo(s.lastActive)}</div>
          <div class="card-actions">
            <button class="action-btn open-btn">Open</button>
            <button class="action-btn danger delete-btn" data-id="${s.id}">Delete</button>
          </div>
        </div>
      `;
    }).join("");

    grid.querySelectorAll(".card").forEach((card) => {
      const el = card as HTMLElement;
      const id = el.dataset.id!;
      const name = decodeURIComponent(el.dataset.name!);

      el.querySelector(".open-btn")!.addEventListener("click", (e) => {
        e.stopPropagation();
        window.location.href = `/app/session.html?id=${id}&name=${encodeURIComponent(name)}`;
      });

      el.querySelector(".delete-btn")!.addEventListener("click", (e) => {
        e.stopPropagation();
        this.deleteSession(id);
      });

      el.addEventListener("click", () => {
        window.location.href = `/app/session.html?id=${id}&name=${encodeURIComponent(name)}`;
      });
    });
  }

  private async deleteSession(id: string): Promise<void> {
    if (!confirm("Delete this session?")) return;
    await fetchWithAuth(`/api/sessions/${id}`, { method: "DELETE" });
    this.fetchSessions();
  }

  private showNewSessionDialog(): void {
    const name = prompt("Session name:");
    if (!name) return;
    const workingDir = prompt("Working directory (leave blank for default):");

    const body: Record<string, string> = { name };
    if (workingDir) body.workingDir = workingDir;

    fetchWithAuth("/api/sessions", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    }).then(() => this.fetchSessions());
  }
}

customElements.define("claudony-session-grid", ClaudonySessionGrid);
```

- [ ] **Step 5: Wire the component into the layout tree**

Update `app/src/main/webui/src/index.ts`:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { hostPanel, rows } from "@casehubio/pages-ui";
import "./components/session-grid";

registerPanel("session-grid", "claudony-session-grid");

const app = rows(
  hostPanel("session-grid"),
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).catch(console.error);
}
```

- [ ] **Step 6: Update index.html with dark theme body styling**

Update `app/src/main/webui/public/index.html` — add body styling so the page isn't white:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="theme-color" content="#007acc">
    <title>Claudony</title>
    <link rel="manifest" href="/manifest.json">
    <style>
      html, body { margin: 0; padding: 0; height: 100%; background: #1e1e1e; color: #ccc; }
      #app { height: 100%; }
    </style>
</head>
<body>
    <div id="app"></div>
    <script type="module" src="app.js"></script>
    <script>
    if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register('/sw.js');
    }
    </script>
</body>
</html>
```

- [ ] **Step 7: Build and verify manually**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server`

Open `http://localhost:7777/auth/dev-login` (POST to get auth cookie), then `http://localhost:7777/app/`. Verify:
- The session grid renders with cards
- Cards show session name, status badge, working dir, time-ago
- Clicking a card navigates to `/app/session.html?id=...` (old terminal page)
- The old terminal page still works
- New Session button prompts and creates a session
- Delete button removes a session

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass. Old `session.html`, `terminal.js`, `style.css` tests still hold.

- [ ] **Step 9: Commit**

```bash
git add app/src/main/webui/src/
git commit -m "feat(#161): claudony-session-grid Web Component via loadSite()

Session grid renders via casehub-pages hostPanel + loadSite().
Fetches /api/sessions every 5s, renders cards with status badges,
working dir, time-ago. Click navigates to old session.html (phase 1).

Refs #161"
```

---

### Task 3: Update StaticFilesTest for Phase 1 Coexistence

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/frontend/StaticFilesTest.java`

**Interfaces:**
- Consumes: Task 1 and 2's Quinoa setup + session grid component
- Produces: Updated test assertions covering both Quinoa-served and old static files

- [ ] **Step 1: Rewrite StaticFilesTest**

Replace the test class with assertions that cover the phase 1 coexistence state:

```java
package io.casehub.claudony.frontend;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class StaticFilesTest {

    // --- Quinoa-served files (new) ---

    @Test
    void quinoaIndexHtmlIsAccessible() {
        given().when().get("/app/index.html")
            .then().statusCode(200)
            .body(containsString("app.js"))
            .body(containsString("<div id=\"app\">"));
    }

    @Test
    void quinoaBundleIsAccessible() {
        given().when().get("/app/app.js")
            .then().statusCode(200)
            .contentType(containsString("javascript"));
    }

    // --- Old files still accessible during coexistence ---

    @Test
    void oldSessionHtmlIsAccessible() {
        given().when().get("/app/session.html")
            .then().statusCode(200)
            .body(containsString("terminal.js"));
    }

    @Test
    void oldTerminalScriptIsAccessible() {
        given().when().get("/app/terminal.js")
            .then().statusCode(200)
            .contentType(containsString("javascript"));
    }

    @Test
    void oldStyleSheetIsAccessible() {
        given().when().get("/app/style.css")
            .then().statusCode(200)
            .contentType(containsString("text/css"));
    }

    // --- PWA / shared assets ---

    @Test
    void manifestJsonIsAccessible() {
        given().when().get("/manifest.json")
            .then().statusCode(200)
            .contentType(containsString("json"))
            .body(containsString("\"name\""))
            .body(containsString("\"start_url\""))
            .body(containsString("standalone"));
    }

    @Test
    void serviceWorkerIsAccessible() {
        given().when().get("/sw.js")
            .then().statusCode(200)
            .contentType(containsString("javascript"))
            .body(containsString("skipWaiting"));
    }

    @Test
    void iconsAreAccessible() {
        given().when().get("/icons/icon-192.svg").then().statusCode(200);
        given().when().get("/icons/icon-512.svg").then().statusCode(200);
    }
}
```

- [ ] **Step 2: Run to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --also-make -Dtest=StaticFilesTest`
Expected: All 8 tests pass.

- [ ] **Step 3: Run full suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass. Test count may decrease slightly (old tests removed, new ones added — net change should be close to zero).

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/claudony/frontend/StaticFilesTest.java
git commit -m "test(#161): update StaticFilesTest for Quinoa coexistence

Quinoa-served files (index.html, app.js) + old files (session.html,
terminal.js, style.css) + shared assets (manifest, sw, icons).

Refs #161"
```

---

### Task 4: Verify Dev Mode Hot Reload and Build

**Files:**
- No new files — verification task

**Interfaces:**
- Consumes: Tasks 1-3
- Produces: Confidence that `mvn quarkus:dev` hot-reloads TypeScript changes, and `mvn package -DskipTests` produces a runnable JAR with the frontend included.

- [ ] **Step 1: Verify dev mode hot reload**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server`

1. Open `http://localhost:7777/app/` (after dev-login)
2. Edit `app/src/main/webui/src/components/session-grid.ts` — change the header text from "Sessions" to "Sessions (dev)"
3. Wait 2-3 seconds for Quinoa rebuild
4. Refresh browser — should show "Sessions (dev)"
5. Revert the change

- [ ] **Step 2: Verify JAR build includes frontend**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests -pl app --also-make
```

Verify the JAR contains the Quinoa output:
```bash
jar tf app/target/quarkus-app/quarkus-run.jar | grep -E "app\.(js|css)" | head -5
```

Expected: `META-INF/resources/app/app.js` should appear in the output (Quinoa copies build output into the JAR's static resources).

- [ ] **Step 3: Test JAR runs and serves the new UI**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) java -Dclaudony.mode=server -jar app/target/quarkus-app/quarkus-run.jar
```

Open `http://localhost:7777/app/` — should show the session grid. Old `session.html` should still work.

Kill the server.

- [ ] **Step 4: Commit (no code changes — verification only)**

No commit needed — this task is verification only.

---

## Self-Review

**Spec coverage check:**
- ✅ Quinoa extension added with correct version
- ✅ `quarkus.quinoa.ui-root-path=/app` for auth protection
- ✅ `quarkus.quinoa.package-manager-install=true` for CI
- ✅ esbuild config from quinoa-host template
- ✅ `.npmrc` with GitHub Packages auth
- ✅ `loadSite()` + `hostPanel()` + `registerPanel()` wired
- ✅ Session grid component replicates existing rendering (cards, status, time-ago, delete, create)
- ✅ Phase 1 coexistence: old `session.html` still works, click navigates to old page
- ✅ `StaticFilesTest` updated for Quinoa-served + old coexistence paths
- ✅ `AppAuthProtectionTest` unchanged (paths still valid)
- ✅ Auth 401 handling in fetch wrapper
- ✅ Dark theme tokens

**Not in phase 1 (deferred to later phases per spec):**
- `stack()` with selector/workbench views (phase 3)
- `pages-event` session-selected / `activateSlot` (phase 3)
- `pages-terminal` stock component (phase 2)
- `/ws/sessions-feed` WebSocket endpoint (phase 4)
- Fleet, mesh, channels, case-workers components (phase 3)
- URL state management (phase 3)
- Old file deletion (phase 5)

**Placeholder scan:** No TBD/TODO found.

**Type consistency:** `Session` interface used consistently. `fetchWithAuth` used in all fetch calls. `timeAgo` imported from same path.
