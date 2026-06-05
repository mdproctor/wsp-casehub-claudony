---
layout: post
title: "The Shell That Outlived the Worker"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub, tmux, concurrency]
---

The problem was simple to state: the engine never found out when a worker finished.
Claude CLI would run, complete, exit — and the case would stay frozen in WAITING
indefinitely. The engine had called `provision()`, gotten a session back, called
`submit()`, and then heard nothing. No completion event, no failure, nothing.

I thought the fix would be a virtual thread that polls `tmux has-session`. Check
every few seconds; when it returns non-zero, the session is gone, publish
`WorkflowExecutionCompleted`. Clean, obvious, done.

Except we had the whole setup wrong. The way Claudony creates tmux sessions —
`new-session` to get a shell, then `send-keys` to type the command — means the
shell stays alive after the command exits. `tmux has-session` doesn't know the
worker finished. It sees the shell, returns zero, and the watcher loops forever.
The session only closes if you're running the command directly, not through a shell.

This is why `createWorkerSession()` exists now. It uses `-- sh -c <command>` and
sets `remain-on-exit off` explicitly, so the session closes when the command exits
regardless of what's in `~/.tmux.conf`. The original `createSession()` is unchanged
for regular user sessions where keeping the shell alive is exactly what you want.

With that fixed, the polling works. But then there's the race: `terminate()` kills
a worker explicitly, and the watcher detects the session gone, and both paths try
to publish completion. The gate we settled on is `registry.remove(sessionId) !=
null` — whichever caller wins that atomic compare-and-remove is the one that
publishes. The other caller gets null back and stays quiet. It also required
flipping the order inside `terminate()`: remove from registry first, then kill the
session. That way if the watcher polls between the two operations, it sees the
registry entry gone and exits cleanly.

The recovery case — server restarts while workers are running — meant persisting
the caseId and role name as tmux session options. `tmux set-option @casehub_case_id`
and `@casehub_role` survive a JVM restart and can be read back on the next boot.
`bootstrapCasehubWatchers()` then fetches the `CaseInstance` from the engine and
restarts the watcher. If the engine has also restarted, there's no CaseInstance
to fetch and the session stays orphaned — that's an accepted limitation for now.

The debugging was mostly concrete. Claude flagged one failure I wouldn't have
caught until runtime: `@Blocking` on `bootstrapCasehubWatchers()`, which I'd added
because the method calls `.await().atMost()` on a reactive type. Correct instinct,
wrong context. `@Blocking` only works in reactive dispatch contexts — JAX-RS,
reactive messaging, event bus consumers. On a plain CDI bean method called from a
startup observer, it causes Quarkus augmentation to fail with a classloader error
that reports the wrong class entirely. The startup observer already runs on the
main thread. The annotation isn't just unnecessary — it breaks things.

The feature works end-to-end now. A worker starts in tmux, runs Claude, exits,
and the engine sees a completion event within the next poll interval. The case
advances. That's been missing since the CaseHub integration began.
