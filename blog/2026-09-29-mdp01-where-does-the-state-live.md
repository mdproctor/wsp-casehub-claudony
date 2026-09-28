---
title: "Where Does the State Live?"
date: 2026-09-29
author: mdp
entry_type: note
subtype: diary
tags: [claudony, agent-pool, tmux, persistence, testing]
projects: [casehubio/claudony]
series: issue-230-agent-pool-test-utilities
---

# Where Does the State Live?

I started this session wanting integration tests for the agent pool — proof that YAML definitions flow through the registry, create real tmux sessions, produce observable output, and clean up after themselves. Claude built seven tests that exercise the chain end-to-end: a shell script mock agent that prints a greeting and echoes stdin, driven through `TmuxSessionOperations` and `AgentSessionManager` with real tmux sessions.

The interesting part wasn't the tests. It was the question they provoked.

We needed a testing module — `claudony-testing` — so downstream projects could test against agent pools without tmux. The obvious implementation: `InMemorySessionOperations`, a test double for the `SessionOperations` SPI. But before building the test double, I wanted to verify the SPI was right. Does it model the right abstractions? Are the persistence boundaries clean?

That's when the problem surfaced. When the pool suspends a session (evicts it to make room), it kills the tmux session and remembers the metadata in a `ConcurrentHashMap`. Identity, working directory, conversation ID — all in memory. If the JVM crashes, the suspended session vanishes without a trace. The system silently loses track of work it promised to manage.

My first instinct was a state file — `~/.claudony/pool-state.json`, written on suspend, removed on resume. Claude pushed back: file-based mutable state is fragile. Race conditions, partial writes, no transactional semantics. Fair point. Database? Over-engineered for a small key-value store.

The answer was already in front of us. tmux IS the persistence layer. Active sessions already prove this — `@claudony_identity` survives JVM crashes because it's a tmux session option. The problem isn't that we lack persistence; it's that suspend *destroys* the persistence layer by killing the session.

The fix: kill the process, not the session. Set `remain-on-exit on`, use `respawn-pane` to resume. The tmux session stays as a metadata container — identity, conversation ID, working directory all preserved as session options. `tmux list-sessions` finds both active and suspended sessions. Active ones have a running process; suspended ones have a dead pane. No file, no database, no new SPI. One source of truth.

This changed how we modelled `InMemorySessionOperations`. The in-memory test double mirrors the production semantics: suspend moves a session from the active map to the suspended map instead of removing it. The session survives suspension. `TestPoolBuilder` wires everything together so downstream tests are one line: `TestPoolBuilder.create().maxActive(5).build()` returns a `TestPool` with both the manager and the inspectable operations.

The architectural insight — that the existing infrastructure already solves your persistence problem if you stop destroying it — is the kind of thing that only surfaces when you ask "is this SPI right?" before building test doubles for it. If I'd built `InMemorySessionOperations` first and verified it against the broken model, we'd have tested the wrong thing and shipped the same fragility.

The suspend model fix is filed as its own issue. The testing module ships now with the correct semantics already baked in. When the production code catches up, the tests will already be proving the right behaviour.
