---
layout: post
title: "Agents Aren't Connections — Why CLI Session Pools Need a Different Model"
date: 2026-09-21
entry_type: note
subtype: diary
projects: [casehubio/claudony]
tags: [agents, fleet-management, session-lifecycle, pool-design]
series: issue-205-llm-fleet-manager
---

# Agents Aren't Connections — Why CLI Session Pools Need a Different Model

I started this session with a plan that treated agent sessions like database connections: capacity-bounded pool, destroy on release, create fresh replacements. The HikariCP model. It compiled, the tests passed, and it was wrong.

The plan had come from a prior design session that modelled the fleet manager as a traditional resource pool. Agents go in, agents come out, the pool manages capacity. Clean abstraction. But when I thought through what actually happens with CLI agents — Claude Code sessions running in tmux — the model fell apart.

A database connection is stateless. You return it, someone else uses it, nobody cares. A Claude session is the opposite. It has a conversation history. It has files it modified. It has a working directory with in-progress work. Destroying it on release destroys all of that context. The next session starts cold — no history, no awareness of what came before. Every interaction is a first date.

## The OS got here first

The right model turned out to be process scheduling, not connection pooling. Active sessions consume resources — a tmux process, memory, a terminal. Suspended sessions consume almost nothing — the conversation ID and working directory persist on disk. Resume is `claude -c <conversation-id>` in the same directory, 1–3 seconds, and the full context comes back.

The state machine is simple: ACTIVE ↔ SUSPENDED. No DESTROYED-and-replaced cycle. Suspend kills the tmux session. Resume creates a fresh one with the conversation flag. The expensive thing — the conversation history, the agent's understanding of its task — survives for free.

## Eviction as a cache, not a timer

The first design had idle timeouts: session unused for 15 minutes, destroy it. But that's the wrong signal. A session idle for 14 minutes isn't less valuable than one idle for 2 minutes — it just hasn't been needed yet. Killing it on a timer means paying the 1–3 second resume cost for no reason.

What actually matters is pressure. If you're running 3 sessions and your ceiling is 10, all 3 stay hot. Only when a new session is needed and you're at capacity does something get evicted. The question then becomes: which one?

Idle time alone isn't enough. A 500MB session idle for 5 minutes is costing more than a 50MB session idle for 10 minutes. The eviction score needs both signals — and they need to be additive, not multiplicative. Multiplicative makes one signal irrelevant when the other is zero: a 500MB session just used scores 0 × 500 = 0. Additive gives it a baseline cost of 50 regardless of recency. The formula we landed on: `idleSeconds + (memoryMB / 10)`. Memory is sampled at the end of each interaction — cheap, no polling loop.

## What this opens up

The session manager now tracks identified instances — each with a conversation ID, identity, and working directory. When a task asks for a "code-reviewer", the manager checks for a suspended reviewer before creating a new one. Resume is cheaper than cold start, and the resumed agent already knows the codebase.

The open question is shared file coordination. The current model assumes one agent per working directory. Debates, parallel review, ensemble critique — patterns where multiple agents touch the same files — need coordination that doesn't exist yet. Branch-per-agent, read-only-input, or explicit locking. Each has trade-offs I haven't worked through. That's the next design problem.
