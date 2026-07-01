---
layout: post
title: "From Closure to Components"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [Claudony]
tags: [web-components, typescript, quinoa, casehub-pages, migration]
---

The terminal page started as 834 lines of JavaScript in an IIFE — five concerns tangled together by closure-scoped state. `sessionId`, `ws`, `chCursors`, `activeCaseId`, `chEventSource` — all living as `var` declarations at the top of a single function, mutated freely by any code path that needed them. It worked. It was also the kind of code where fixing the channel panel meant reading the worker panel to understand what state it might have changed.

The migration to TypeScript components forced a different structure. Five components, each owning its own state, communicating via `pages-event` CustomEvents. The channel panel doesn't know the worker panel exists. The worker panel doesn't know about channels. The workspace coordinator routes events between them — `worker-selected` triggers terminal reconnection, channel reconfiguration, and header update, but no component reaches into another's internals.

The design review surfaced things I wouldn't have caught by reading the code. The spec's minimal `session.html` silently dropped the xterm.css CDN link — PagesTerminal doesn't bundle its own CSS. The compose overlay was calling `sendInput()` when it needed `paste()` — the difference is bracketed paste mode, which prevents multi-line input from being executed line-by-line by zsh. The proxy peer URL construction was specified for the WebSocket but not for the resize API call. All of these would have been silent runtime failures: no error, just wrong behaviour.

Claude caught one more during the final code review: `ClaudonyTerminalHeader.configure()` was replacing the entire `innerHTML` on every call, which destroyed the event listeners the entry point had wired to the Workers, Channels, and Compose buttons. After a worker switch, all three buttons went dead — no error, no visual change, just inert HTML. The fix was obvious once named: check whether the DOM already exists, and if so, update text nodes instead of re-rendering. But the failure mode was genuinely hard to spot in testing because it only triggered on the second `configure()` call, not the first.

The channel panel — at 1062 lines of TypeScript — is now the largest single component in the frontend. It handles SSE with polling fallback, stale cursor detection via `sessionStorage`, case context headers with elapsed-time tickers, expandable lineage displays, and per-channel message type filtering. Every DOM selector that the 19 Playwright E2E tests depend on was preserved. The port was faithful by design: the E2E tests are the contract, and breaking them would have meant debugging behaviour differences with no error messages to guide the search.

esbuild's code splitting means the dashboard bundle doesn't load xterm.js, and the terminal bundle doesn't load the session grid. Shared dependencies — `pages-runtime`, `pages-ui`, theme, auth utilities — land in a common chunk that both pages reference. Two entry points, one `npm run build`, no duplication.

The worker panel and channel panel were built as reusable components for a reason. Issue #75 — the three-panel unified dashboard — needs both of them. When that work starts, the panels are ready to compose into a different layout without porting any logic.
