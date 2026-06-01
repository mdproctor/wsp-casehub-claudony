---
layout: post
title: "Three layers to one line"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [engine, testing, tenancy, quarkus]
---

I came in expecting to wait the engine round-trip test out. The previous session left a note: `CaseEngineRoundTripTest` fails because the engine SNAPSHOT has an in-flight `PlanExecutionContext` constructor change. Engine problem. Sit tight.

I wasn't convinced. I brought Claude in to look at it fresh — run the build, read the actual error, diagnose from scratch.

That note was wrong.

The error was a `ConditionTimeoutException` from Awaitility. Lineage was empty after ten seconds. Buried at the bottom of the output, marked `Suppressed:`, was a `NullPointerException: Cannot invoke CaseInstance.getCaseContext() because caseInstance is null` inside `WorkflowExecutionCompletedHandler`. Not a constructor change. An NPE swallowed by the Vert.x event bus.

We traced it back. The test builds the round-trip by looking up the case instance, then publishing `WorkflowExecutionCompleted` to the engine's event bus. If the lookup returns null, the event goes out with a null `caseInstance`. The handler NPEs. The lifecycle event that was supposed to follow — `WorkerExecutionCompleted` — never fires. `ClaudonyLedgerEventCapture` never writes anything. Lineage stays empty.

The lookup was `caseInstanceRepository.findByUuid(caseId, "default")`. The engine SNAPSHOT added `tenancyId` filtering to the new two-argument signature. We decompiled `InMemoryCaseInstanceRepository` from the local Maven JAR to read the actual bytecode.

The implementation does `tenancyId.equals(instance.tenancyId)`. When the engine stores a case with `tenancyId=null` — which it does when the test profile has no tenancy configured — `"default".equals(null)` returns false. The method returns `Uni.nullItem()`. The UUID is in the store. The method returns null anyway, with no log, no warning, nothing.

Three layers of indirection hiding a single mismatched parameter. The previous session anchored on the `PlanExecutionContext` change because it was visible in the SNAPSHOT notes and looked plausible. A reasonable guess. Wrong.

The fix is one import and two line changes. `InMemoryCaseInstanceRepository` also implements `CrossTenantCaseInstanceRepository`, which has `findByUuid(UUID)` — no tenancy parameter. That's the right API for a test with no tenancy context. 532 of 532 passing.

---

Fixing that test raised a question I'd been deferring: if the engine round-trip works now, how close are real-world examples?

Closer than I expected. Two blockers, not ten.

The first is `WorkerExecutionManager`. The engine schedules workers via Quartz and needs something to close the loop — detect when the Claude process in a tmux session finishes and publish `WorkflowExecutionCompleted`. The test fires this manually. Production has nothing. The outline is a virtual thread watcher in the provisioner: poll `tmux has-session`, publish on exit. Not complicated, just not written.

The second is a real `CaseDefinition`. `TestResearcherCase` is test scaffolding and doesn't exist at runtime. One concrete CDI bean following the same pattern is enough to have a workflow you can call `startCase()` on and watch end to end.

Everything else is working. Channels, SSE delivery, ledger capture, mesh participation, authentication. What's missing are the two ends: something that knows when a worker finishes, and something that defines what a case is.
