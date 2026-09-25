---
title: "Three Frontends, One Model"
date: 2026-09-25
author: mdp
entry_type: note
subtype: diary
series: issue-231-canonical-pool-layer
projects: [casehubio/claudony, casehubio/platform]
tags: [agent-pool, dsl, annotations, yaml, process-executor, desiredstate]
---

The canonical layer problem in the casehub ecosystem is always the same: you have a raw Java API that works, and you need to make it declarative. The pattern — fluent DSL, annotations, YAML — is established across engine, blocks, work, and desiredstate. The agent pool didn't have it.

I built all three in a single session. `AgentPoolDefinition` is an immutable value type with nested builder scopes — `.agent("code-reviewer").workingDir("/reviews").pool().minActive(2).maxActive(10).build()`. The annotations (`@PooledAgent`, `@AgentPool`) are sugar over the builder — the scanner reads them via reflection and calls the same DSL. The YAML parser does the same with Jackson. Three frontends, one model type, equivalence proven by tests that build the same definition both ways and assert equality.

The interesting part came after. Issue #234 was scoped as "ops YAML plugin integration" — wire the pool into ops's plugin system. But the design conversation went somewhere I didn't expect. The ops YAML plugin architecture has a three-layer stack: Java SPIs at the bottom (RestClient, AuthProvider), YAML primitives in the middle (rest-call, paginate, poll-until), YAML plugins at the top composing over both. Every Layer 2 primitive is HTTP-oriented. The pool needs local process management — tmux commands. There's no `process-call` primitive because there's no `ProcessExecutor` SPI for it to compose over.

That gap isn't pool-specific. Any CLI-managed resource — docker, terraform, ansible — hits the same wall. And Claudony already has 30 ProcessBuilder call sites, all reimplementing the same inline pattern with no timeouts, no structured output capture, no error classification.

So I built `ProcessExecutor` in `platform-api`. Five classes: the SPI interface, a ProcessBuilder-backed implementation with async stream capture and timeout enforcement, an immutable command descriptor, a structured result type, and an exception. It's now on platform's local main, available for yaml-core to build a `process-call` primitive on top of, and for Claudony to migrate its 30 call sites onto.

The other question that surfaced: should pooling be expressed as desiredstate? A pool has a desired state (min 2, max 10 sessions) and an actual state (currently 1 active). The reconciliation loop's job is making actual match desired — which is what pool pre-warming does. But pool operations are request-driven (acquire a session now), not periodic (reconcile every 5 minutes). The pool's internal lifecycle belongs to Claudony's runtime, not to the reconciliation loop. The provisioner is the thin bridge between them.
