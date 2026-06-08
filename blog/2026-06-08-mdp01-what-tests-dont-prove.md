---
layout: post
title: "What Tests Don't Prove"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub, binding, yaml, quarkus]
---

# Claudony — Running the Signal Chain Live

## What I was trying to prove: the tests told us it worked, but the server would

The previous phase proved the ResearcherCase signal chain in tests — call `startCase()`, the exit signal fires, context patches, case reaches COMPLETED. But tests mock the tmux watcher; they never create a real session. Until we can trigger a case from a browser and watch a Claude window appear, all we have is a proof-of-concept in a harness.

Three things were missing. `casehub-engine` is test-scope, so the production CDI configuration explicitly excludes `ResearcherCase` — correct for production, but too aggressive for dev mode where the engine is available. There was no REST endpoint to trigger `startCase()`. And the dev profile had no CaseHub config at all.

## The binding problem we almost missed

The YAML change looked straightforward: replace the `.topic != null` filter with something that fires immediately on case start. The engine has no dedicated StartTrigger, but the mapper converts a `contextChange` with no filter into a `ContextChangeTrigger(null)` — and a null evaluator evaluates to true, so the binding fires unconditionally on any context change.

What we didn't think about was the second context change.

When the tmux session exits, `CaseHubRuntime.signal("workers.researcher.exited", true)` patches the context. That's another `CONTEXT_CHANGED` event. And `ChoreographyLoopControl` — the active default — checks exactly one thing before passing bindings through: `caseStatus == RUNNING`. At the moment the exit signal fires, the case is still `RUNNING`; the goal and completion transitions are async. So the null-filter binding fires a second time, a second session is provisioned, and by the time the case reaches COMPLETED there's an unwanted Claude window alive.

The fix is the `when:` guard. After the trigger filter, `CaseContextChangedEventHandler.rules()` evaluates `binding.getWhen()`. On the initial empty context: `.workers.researcher.exited` is null, `null != true` is true, binding fires. On the exit signal patch: `.workers.researcher.exited` is true, `true != true` is false, binding skipped. One line of YAML, but the absence of it would have been a confusing bug to track down live.

We caught it in review, before anything ran.

## The YAML syntax trap

The null-filter YAML form had its own surprise. The natural way to write a contextChange binding with no filter is bare `contextChange:` — no value. In YAML, no value is null. `CaseDefinitionYamlMapper.convertTrigger()` calls `trigger.getContextChange()` and immediately branches on null — not to "no filter", but to `UnsupportedOperationException: Only ContextChangeTrigger is currently supported`.

`contextChange: {}` (empty map) produces a non-null model object with a null filter field. That's the form the mapper handles correctly. The error message points you at the trigger type, not the YAML value — an easy rabbit hole.

## The endpoint

The REST side was the most mechanical part. `CasehubResource` at `/api/casehub/cases/researcher` uses `@Inject Instance<ResearcherCase>` — the same pattern `ServerStartup` already uses for `ClaudonyWorkerExecutionManager`. If the engine isn't in CDI, `isUnsatisfied()` is true and the endpoint returns 503. If it is, `startCase()` goes. The return type is `CompletionStage<Response>` rather than `@Blocking` — RESTEasy Reactive suspends the request and resumes on completion without holding a thread, which is the right fit since `startCase()` itself returns a `CompletionStage<UUID>`.

The dev profile adds three lines: `%dev.quarkus.arc.exclude-types` (the same ledger conflict beans, minus `ResearcherCase`), `%dev.claudony.casehub.enabled=true`, and `%dev.claudony.casehub.workers.commands.researcher=claude`. The `%dev` value replaces the base list, not appends to it — which is why you have to repeat the three ledger beans that still need excluding.

Two garden entries and two protocols came out of this work. The `when:` guard requirement is now a protocol: any null-filter binding must carry a guard expression that excludes re-fire conditions, because the engine won't do it for you.
