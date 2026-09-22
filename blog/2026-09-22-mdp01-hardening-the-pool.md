---
layout: post
title: "Hardening the Agent Pool — From Design to Deployable"
date: 2026-09-22
entry_type: article
subtype: diary
projects: [casehubio/claudony]
tags: [agents, fleet-management, pool-design, audit, downstream-integration]
series: issue-205-llm-fleet-manager
---

# Hardening the Agent Pool — From Design to Deployable

*Continues from [Agents Aren't Connections](2026-09-21-mdp01-agents-arent-connections.md), which covered why CLI agent sessions need suspend/resume semantics instead of connection pooling.*

The design was right. The implementation had holes.

The last session landed a working agent pool — capacity-bounded, suspend/resume lifecycle, pressure-based eviction, identity-correlated instances. It compiled, the tests passed. But "it works on the happy path" is not the same as "it works." An adversarial audit found seven issues, three of them correctness bugs that would have silently degraded the pool in production.

## The pool leak nobody would have noticed

The most dangerous bug was in `close()`. When a worker finishes its task, the CaseHub engine calls `close()` on the agent session. The implementation recorded the final memory interaction — dutifully updating the eviction metadata — then returned. It never released the session back to the pool.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 200" style="max-width:700px;font-family:-apple-system,system-ui,sans-serif">
  <defs>
    <marker id="ah1" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#666"/></marker>
  </defs>
  <text x="20" y="30" font-size="13" font-weight="bold" fill="#e74c3c">Before — the leak</text>
  <rect x="20" y="45" width="120" height="36" rx="6" fill="#2ecc71" stroke="#27ae60"/>
  <text x="80" y="68" text-anchor="middle" font-size="12" fill="white">ACTIVE</text>
  <line x1="140" y1="63" x2="220" y2="63" stroke="#666" marker-end="url(#ah1)"/>
  <text x="180" y="55" text-anchor="middle" font-size="10" fill="#666">close()</text>
  <rect x="225" y="45" width="120" height="36" rx="6" fill="#2ecc71" stroke="#27ae60"/>
  <text x="285" y="68" text-anchor="middle" font-size="12" fill="white">still ACTIVE</text>
  <text x="225" y="105" font-size="11" fill="#e74c3c">↑ slot consumed forever</text>

  <text x="20" y="140" font-size="13" font-weight="bold" fill="#27ae60">After — the fix</text>
  <rect x="20" y="155" width="120" height="36" rx="6" fill="#2ecc71" stroke="#27ae60"/>
  <text x="80" y="178" text-anchor="middle" font-size="12" fill="white">ACTIVE</text>
  <line x1="140" y1="173" x2="220" y2="173" stroke="#666" marker-end="url(#ah1)"/>
  <text x="180" y="165" text-anchor="middle" font-size="10" fill="#666">close()</text>
  <rect x="225" y="155" width="120" height="36" rx="6" fill="#f39c12" stroke="#e67e22"/>
  <text x="285" y="178" text-anchor="middle" font-size="12" fill="white">SUSPENDED</text>
  <text x="225" y="210" font-size="11" fill="#27ae60">↑ slot freed for reuse</text>
</svg>

Every worker session that completed leaked one active slot. In a production scenario with repeated case execution — a research pipeline running 20 cases a day — the pool would exhaust its capacity ceiling within hours. No error, no warning. New workers would simply fail to provision with `AgentPoolExhaustedException`, and the operator would be staring at a full pool of sessions that are all "active" but doing nothing.

The fix is one line: `sessionManager.suspendSession(managedSession.instanceId())` at the end of `close()`. The session transitions to SUSPENDED, the active slot is freed, and the conversation state persists on disk for potential resume. One line, but the kind of line that separates "demo" from "deployable."

## The eviction formula that didn't match the design

The design spec said eviction should be multiplicative — memory amplifies idle time. The implementation was additive: `idleSeconds + (memoryMB / 10)`. This meant a 500MB session idle for 1 second scored 51, while a 10MB session idle for 50 seconds scored 51 too. Memory barely registered for short idle periods.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 240" style="max-width:700px;font-family:-apple-system,system-ui,sans-serif">
  <text x="20" y="25" font-size="13" font-weight="bold" fill="#333">Additive: idle + mem/10</text>
  <rect x="20" y="35" width="300" height="24" rx="4" fill="#3498db"/>
  <text x="170" y="52" text-anchor="middle" font-size="11" fill="white">10MB, 50s idle → score 51</text>
  <rect x="20" y="65" width="306" height="24" rx="4" fill="#e74c3c"/>
  <text x="173" y="82" text-anchor="middle" font-size="11" fill="white">500MB, 1s idle → score 51</text>
  <text x="340" y="77" font-size="11" fill="#666">← tied (memory barely matters)</text>

  <text x="20" y="130" font-size="13" font-weight="bold" fill="#333">Multiplicative: (idle+1) × (1 + mem/100)</text>
  <rect x="20" y="140" width="280" height="24" rx="4" fill="#3498db"/>
  <text x="160" y="157" text-anchor="middle" font-size="11" fill="white">10MB, 50s idle → score 56</text>
  <rect x="20" y="170" width="72" height="24" rx="4" fill="#e74c3c"/>
  <text x="56" y="187" text-anchor="middle" font-size="11" fill="white">score 12</text>
  <text x="100" y="187" font-size="11" fill="#666">500MB, 1s idle</text>
  <text x="20" y="215" font-size="11" fill="#27ae60">← memory amplifies, doesn't replace idle time</text>
</svg>

The multiplicative formula — `(idleSeconds + 1) × (1 + memoryMB / 100)` — preserves idle time as the primary signal while giving memory-heavy sessions a proportional penalty. The `+1` base ensures a just-created 500MB session doesn't score zero. A session consuming 500MB of RAM is five times more expensive to keep around than a 10MB session, all else being equal — and the formula now reflects that.

## Making the pool consumable

Fixing bugs is necessary. Making the pool usable by other projects is the actual goal. Claudony isn't the only project that needs to manage agent sessions — scaffold, desiredstate, and ops all provision workers, and they're about to integrate with this pool.

The pool needed three things before downstream projects could use it: documentation, CDI injectability, and API clarity.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 300" style="max-width:700px;font-family:-apple-system,system-ui,sans-serif">
  <text x="350" y="25" text-anchor="middle" font-size="14" font-weight="bold" fill="#333">Agent Pool — Consumer Surface</text>

  <rect x="220" y="40" width="260" height="50" rx="8" fill="#2c3e50"/>
  <text x="350" y="60" text-anchor="middle" font-size="13" fill="white" font-weight="bold">ClaudonyAgentBackend</text>
  <text x="350" y="77" text-anchor="middle" font-size="10" fill="#bdc3c7">@ApplicationScoped · @Produces AgentSessionManager</text>

  <line x1="350" y1="90" x2="350" y2="120" stroke="#666" stroke-dasharray="4" marker-end="url(#ah1)"/>

  <rect x="200" y="125" width="300" height="50" rx="8" fill="#34495e"/>
  <text x="350" y="145" text-anchor="middle" font-size="13" fill="white" font-weight="bold">AgentSessionManager</text>
  <text x="350" y="162" text-anchor="middle" font-size="10" fill="#bdc3c7">acquire · suspend · resume · destroy · evict</text>

  <rect x="50" y="210" width="160" height="40" rx="6" fill="#8e44ad" stroke="#7d3c98"/>
  <text x="130" y="235" text-anchor="middle" font-size="11" fill="white">scaffold</text>

  <rect x="270" y="210" width="160" height="40" rx="6" fill="#8e44ad" stroke="#7d3c98"/>
  <text x="350" y="235" text-anchor="middle" font-size="11" fill="white">desiredstate</text>

  <rect x="490" y="210" width="160" height="40" rx="6" fill="#8e44ad" stroke="#7d3c98"/>
  <text x="570" y="235" text-anchor="middle" font-size="11" fill="white">ops</text>

  <line x1="130" y1="210" x2="280" y2="175" stroke="#8e44ad" stroke-dasharray="4"/>
  <line x1="350" y1="210" x2="350" y2="175" stroke="#8e44ad" stroke-dasharray="4"/>
  <line x1="570" y1="210" x2="420" y2="175" stroke="#8e44ad" stroke-dasharray="4"/>

  <text x="350" y="280" text-anchor="middle" font-size="11" fill="#666">@Inject AgentSessionManager — one pool, three consumers</text>
</svg>

**CDI exposure** was the critical piece. The `AgentSessionManager` was created internally by `ClaudonyAgentBackend`'s constructor — invisible to CDI. A downstream project wanting to acquire or release sessions had no injection point. A `@Produces @ApplicationScoped` method on the backend now makes the session manager injectable. One annotation, but it changes the pool from an implementation detail to a platform capability.

**The consumer guide** got a full Agent Pool Management section: lifecycle state machine, key classes table, eviction explanation, REST endpoint reference, CDI injection example, and config properties. A developer integrating the pool from scaffold can read the guide and know exactly what to inject, what methods to call, and what config knobs to tune.

## Where this lands next

Each downstream project uses the pool differently:

**scaffold** provisions agent sessions for template generation — short-lived, one-shot tasks. It needs acquire/release with EXCLUSIVE working directory policy. The sessions are cheap, the pool turnover is high, and the default eviction formula works without tuning.

**desiredstate** runs reconciliation loops — long-lived agents that monitor and correct system state. These sessions are identity-sticky: a reconciler for the auth subsystem should resume with its prior conversation context, not start cold. The pool's identity-correlated resume (match by identity + workingDir before creating new) handles this directly.

**ops** is the monitoring layer — it needs pool status (`GET /api/agent-pools`) to surface health in the operational dashboard. It also provisions diagnostic agents that investigate alerts, which means concurrent sessions on the same working directory under SHARED_READ policy.

The plan is to spike scaffold first. Try to integrate, discover which abstractions are right and which are wrong, then fix the pool with evidence before desiredstate and ops arrive. Two deferred issues track the gaps that might surface: a pluggable `EvictionPolicy` SPI (if any consumer needs custom scoring) and a test utilities JAR (once the downstream test pattern is clear).

I was wrong about the eviction formula in the first design session — additive instead of multiplicative. I was wrong about `close()` — it recorded metrics without releasing the slot. The audit found both. The lesson isn't "audit everything" — it's that the difference between a working design and a deployable one is the gap between the happy path and every other path. The pool works now. The next test is whether scaffold agrees.
