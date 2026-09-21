---
title: "Killing dashboard.js"
date: 2026-07-30
entry_type: note
subtype: diary
status: draft
projects: [casehub-claudony]
tags: [lit, web-components, pages-ui-components, migration]
---

The claudony frontend had a split personality. Half of it was modern TypeScript — Lit components, esbuild, design tokens. The other half was `dashboard.js`: 800 lines of vanilla JS doing innerHTML string concatenation, manual event wiring, and hardcoded hex colours. Every time I looked at the session grid I'd think "this needs to go" and then move on to something more urgent.

Issue #185 was the forcing function. casehub-pages shipped `@casehubio/pages-ui-components` — standalone Lit web components for buttons, inputs, selects, textareas, checkboxes. The whole point: consumer apps like claudony should use them instead of native HTML elements with hand-rolled CSS. But you can't just swap `<button>` for `<pages-button>` in an innerHTML template string. The components need a Lit render context. Which means the vanilla JS has to go.

## What we actually did

Three things, in sequence.

**Built missing pages components.** The pages library had form inputs but no badge, status dot, or compact button size. We added `<pages-badge>` (semantic status pills — success/warning/danger/neutral/info/accent), `<pages-status-dot>` (8px coloured circles for worker/health indicators), and `<pages-button size="xs">` (11px compact variant for dock strips and action bars). Filed as casehub-pages#255. Also got pages to ship pre-built static assets — a CSS theme file and a self-registering JS bundle — so login and register pages can use components without a build step. That was casehub-pages#247.

**Decomposed dashboard.js into three typed LitElement components.** The fleet panel (~120 lines of peer management) became `claudony-fleet-panel.ts`. The mesh panel (~370 lines of overview/channel/feed views with SSE/polling) became `claudony-mesh-panel.ts`. The session grid absorbed the remaining dashboard.js features — git/PR status, service health checks, iTerm2 integration, new session dialog (now `<pages-modal>`), auth overlay. Five existing components (terminal-header, worker-panel, key-bar, terminal-workspace, session-grid) converted from vanilla HTMLElement to LitElement.

**Wired the remaining blocks-ui channel-activity components.** Added thin REST endpoints in `MeshResource` for Qhorus reactions, topics, and channel membership — all delegating to existing services that were already there but unexposed. Then wired `<blocks-channel-reaction-bar>`, `<blocks-channel-topic-bar>`, `<blocks-channel-member-panel>`, and `<blocks-channel-thread>` into the workbench. Thread view is pure client-side grouping — messages already had `inReplyTo`, we just needed to render the tree.

## The design review catch

Before implementation, we ran an adversarial design review. The reviewer caught that the spec's reaction endpoint didn't match the actual `ReactionService` API — it proposed a channel-scoped GET but the service only has per-message and batch-by-message-ID methods. Also caught that the spec claimed "no new data models — Qhorus types returned directly as JSON" which was wrong — `ChannelMembership` and `TopicSummary` fields don't match the blocks-ui TypeScript types. Both needed adapter functions in `channel-adapter.ts`. Eleven issues raised, all verified and fixed before we wrote a line of implementation code.

## The platform question

The interesting part wasn't the migration itself — it was the moment we asked "should pages ship pre-built static assets?" That shifted the work from a claudony-specific hack (add an `auth.ts` entry point for login/register) to a platform capability (every Quarkus consumer gets components in static HTML for free). One issue filed, solved the problem for every future consumer. The best code is the code you don't write in your own repo.
