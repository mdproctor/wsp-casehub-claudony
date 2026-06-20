# Handoff — 2026-06-21

**Head commit (project):** `061c013` — chore(claudony#150,#154): rename researcher→agent; close #154

## What landed this session

### claudony#151 — Context.isOnEventLoopThread() threading fix

Engine `a6620c03` added `@ConsumeEvent(blocking=true)` to `CaseContextChangedEventHandler`. Vert.x `executeBlocking` workers inherit the parent event loop Context, so `isEventLoopContext()` incorrectly returns `true` for them. `@WithSession`'s `runSubscriptionOn` uses Hibernate Reactive's `VertxContext.execute()` which asserts the actual thread — throws HR000068 on executeBlocking workers.

Fix: `Context.isOnEventLoopThread()` (public Vert.x 4.x static API) in:
- `ClaudonyReactiveCaseChannelProvider.listChannels()` — guards `doListChannels()` (@WithSession)
- `JpaCaseLineageQuery.findCompletedWorkers()` — avoids `runSubscriptionOn(workerPool)` on worker threads
- `ClaudonyReactiveWorkerContextProvider.buildContext()` — `emitOn(stableEventLoopContext)` guard

Protocol PP-20260620-cb7137 captures this. All 587 tests green, CI confirmed.

### claudony#150 — researcher→agent rename

`researcher` implied academic research; actual capability is generic tmux session management. Renamed:
- `researcher.yaml` → `agent.yaml`; context paths `.workers.researcher.*` → `.workers.agent.*`
- `ResearcherCase` → `AgentCase`; REST: `/api/casehub/cases/agent`; config: `...workers.commands.agent`
- 46 files changed; all test classes renamed; 587 tests all green

### claudony#154 — ResearcherCaseCompletionTest (closed, no code change)

Was already passing after #151 fix. Renamed to `AgentCaseCompletionTest` as part of #150.

## Open threads

- **parent#291**: `docs/repos/claudony.md` deep-dive needs REST path + capability name updated for #150. Must be done in a parent repo session.
- CI queued for `061c013` — expected green (verified locally).

## State

- main: `061c013`
- All 587 tests pass locally
- Branch `issue-154-155-sweep` stamped closed

## Next candidates

No more S/XS issues open. Next M-scale: #105 (separate Claudony/Qhorus MCP endpoints, Phase A architecture).
