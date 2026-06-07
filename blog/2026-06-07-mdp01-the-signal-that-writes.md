---
layout: post
title: "The Signal That Writes"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub, case-completion, signal, exit-watcher]
---

The question I couldn't answer going into this: how does a Claude Code session tell the case engine it's done?

The researcher case in Claudony is a tmux session running Claude Code. When it exits, the exit watcher detects the vanished session and publishes a `WorkflowExecutionCompleted` event. That tells the engine a worker finished. It says nothing about whether the case should close. That requires a goal to be satisfied. Which requires context to change. Which means someone has to write something.

My first instinct was an MCP tool. Claude calls it when done, it signals completion. Neat, but it changes Claude's workflow — the session needs to know about CaseHub, the system prompt needs instruction about when to call it. I wanted zero Claude-side changes.

Then workspace files came up. Claude writes a result to a known path; the exit watcher reads it. Cleaner, but now you have a file path convention that has to live somewhere, be explained in the prompt, and be reliable across all worker types.

What I didn't know: `CaseHubRuntime.signal(caseId, path, value)` isn't named right.

I assumed "signal" meant "fire an event with this name and payload." That's what signal-dispatch APIs do. Wrong. What `signal()` does is write `value` to the case context at the dot-notation `path`, then fire `CONTEXT_CHANGED`. Look at the source and you see it immediately: `CaseContextImpl.applyAndDiff(path, value)` splits on `.` and creates nested maps. `signal(caseId, "workers.researcher.exited", true)` sets `context.workers.researcher.exited = true` and triggers goal evaluation.

So the YAML case definition gets:

```yaml
goals:
  - name: research-complete
    condition: ".workers.researcher.exited == true"
    kind: success
completion:
  success:
    allOf:
      - research-complete
```

And the exit watcher writes to the map. No protocol, no Claude awareness, no file conventions.

The pattern mirrors `drainCausalContext` (from the provisioner). When the watcher detects exit, it stores `pendingExitSignals.put(caseId, roleName)` — before publishing `WorkflowExecutionCompleted`. Then `ClaudonyLedgerEventCapture`, observing the downstream `WorkerExecutionCompleted` lifecycle event, drains it and fires `signal("workers." + roleName + ".exited", true)`.

The ordering constraint was real and got caught in code review. Call `signal()` directly from the watcher thread right after `eventBus.send()` and you're racing: `SignalReceivedEventHandler` may process the context write before `WorkflowExecutionCompletedHandler` has updated engine state. Put the drain before `em.flush()` in the ledger observer and the engine's reaction to `CONTEXT_CHANGED` may query the ledger before the row exists.

Put before send, drain after flush. Two constraints, both now in the protocols.

Getting the E2E test to work surfaced two surprises. First: `CasehubEnabledProfile` (the profile that enables the full engine in tests) excludes `CaseStatusChangedHandler`. That handler is what actually writes `COMPLETED` to `CaseInstance.state`. Without it, goals are satisfied, the logs say `Goal research-complete REACHED`, and the case stays `RUNNING` indefinitely. Second: `DefaultWorkerExecutionRecoveryService` is a non-obvious dependency of `SignalReceivedEventHandler`. Remove one without the other and CDI deployment fails — not at the signal handler, at an unsatisfied `WorkerExecutionRecoveryService` injection point.

The production `ResearcherCase` extends `YamlCaseHub`, loads `casehub/researcher.yaml` from classpath, and is registered at startup. The first real case definition shipping with Claudony. When a Claude Code session exits naturally, the case knows it.
