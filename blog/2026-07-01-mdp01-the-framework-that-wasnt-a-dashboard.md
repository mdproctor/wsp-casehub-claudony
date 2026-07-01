---
layout: post
title: "The Framework That Wasn't a Dashboard"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [Claudony]
tags: [casehub-pages, quinoa, frontend, web-components, design-review]
series: issue-161-adopt-casehub-pages-quinoa
---

The issue said "Scale: S — convention + config + migrate existing grid." Two days later there are twelve commits on the branch, three upstream SNAPSHOT adaptations, an adversarial design review, and the build is blocked on a cross-repo npm publish. The scale was wrong. The complexity was wrong. The one thing that was right: this needed to happen.

Claudony's frontend is 2600 lines of vanilla JavaScript — `dashboard.js`, `terminal.js`, `style.css`. Hand-coded DOM manipulation, polling loops, inline event wiring. It works. But every future UI issue (#75 three-panel dashboard, #84 project setup wizard, #85 agent onboarding) would add another few hundred lines of the same pattern. casehub-pages exists to replace that pattern with a component model and layout engine.

The brainstorming started from a wrong premise. Claude's exploration pulled stale documentation and concluded that casehub-pages was a data visualization DSL — tables, charts, lookups. I corrected this: pages is a web application framework. `hostPanel()` mounts arbitrary Web Components. `split()` creates resizable panel layouts. `stack()` switches between whole-screen views. The session grid, terminal, channel panel — these aren't pages DSL components, they're custom Web Components hosted inside a pages layout tree. Pages manages the shell; Claudony provides the panels.

The design review surfaced real problems. The original spec used `pages-filter` for view switching — clicking a session would fire a cross-filter event to switch from the selector view to the workbench. The reviewer pointed out that `pages-filter` carries `columnId`, `value`, `row` semantics — it's a data-filtering mechanism, not a navigation mechanism. The fix: `pages-event` (a plain DOM custom event bus) combined with `activateSlot()` for stack transitions.

The reviewer also caught that `SessionRegistry.addChangeListener()` is inadequate for the planned `/ws/sessions-feed` WebSocket endpoint. The existing listener passes `caseId` not `sessionId`, doesn't fire on `register()` or `touch()`, and fires `remove()` notifications before the session is actually removed. The spec now specifies a typed `SessionEvent` API.

The most satisfying discovery: casehub-pages had already built a stock terminal component (`@casehubio/pages-component-terminal`). I'd filed an issue for it; by the time I looked, it was done. URL template substitution for `{cols}/{rows}`, exponential backoff reconnect, `terminal.reset()` before reconnect (correct for Claudony's server-side history replay), close code 4001 for session expiry instead of in-band JSON. The design review for the terminal had independently caught the same collision risk I'd specified — echo `'{"type":"session-expired"}'` in a terminal and the component would swallow it as a control message. The fix was to move expiry signalling out of band entirely. One-line change on the server side.

Three upstream SNAPSHOT breaks hit during implementation. The engine changed `Worker.Builder.capabilities(List<Capability>)` to `capabilityName(String)`. Qhorus moved `Channel` and `Message` from runtime POJOs to API-package records with builders. Both broke compilation and CDI deployment across multiple test profiles — 50 deployment errors resolved across three passes, each fix revealing the next layer. The third break was Quinoa itself: the extension is disabled in `@QuarkusTest` mode, hardcoded in `QuinoaProcessor` with no config override. Tests that verify Quinoa-served files get 404. The workaround is `@QuarkusIntegrationTest`.

The branch is functionally complete — Quinoa wired, session grid rendering via `loadSite()`, tmux blank-line test re-architected, all upstream breaks resolved. Blocked on one thing: casehub-pages needs to publish the new npm packages so the `file:` local dependencies can switch to registry versions and CI works.

The terminal component adoption is the pattern that matters here. Claudony needed a terminal. Rather than building one, it drove the requirement upstream — and the upstream implementation came back better than the spec. The resize protocol is server-agnostic (events, not REST calls). The close code 4001 convention is cleaner than in-band JSON. This is how the platform is supposed to work: apps drive requirements, the framework evolves, everyone benefits.
