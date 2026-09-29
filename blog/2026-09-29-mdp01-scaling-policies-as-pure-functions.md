---
title: "Scaling Policies as Pure Functions"
date: 2026-09-29
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/claudony]
series: feat/206-auto-scaling-policies
tags: [agent-pools, scaling, spi, design]
---

The agent pool system had static capacity — `minActive` and `maxActive` set at startup, never adjusted. That's fine when load is predictable. It's not fine when you have pools of Claude sessions that spike during case execution and go idle between cases. The question was whether scaling policies could be a runtime configuration, not a code change.

The answer turned out to be yes, and the design that fell out was cleaner than I expected.

## The key insight: scaling adjusts the ceiling, not the population

I initially assumed scaling meant proactively creating sessions ahead of demand — the AWS auto-scaling model where you pre-warm instances. But Claudony's sessions are identity-correlated. Each session belongs to a specific agent identity and working directory. You can't pre-create anonymous sessions and assign them later.

So scaling became reactive: the policy adjusts `effectiveMaxActive` — a mutable ceiling that sits below the immutable hard ceiling from YAML config. When demand rises, the ceiling rises. When demand drops, the ceiling drops. Sessions are still created on-demand via `acquireSession()` — the policy just controls how much room they have.

This gives the right feedback dynamics. A target-tracking policy that maintains 70% fill ratio works because increasing `maxActive` *decreases* the fill ratio (denominator grows, numerator stays). No positive feedback loop. The system converges in one step.

## Pure functions with external scheduling

We built `ScalingPolicy` as a pure function: `evaluate(PoolSnapshot) → ScalingDecision`. The policy receives an immutable snapshot of pool state — active count, suspended count, min/max, demand metrics — and returns a decision: scale out by N, scale in by N, or do nothing.

The scheduler handles everything else: cooldown enforcement, policy caching, applying decisions to the manager. The policy doesn't know about time, doesn't track state, doesn't call CDI. It's a function from numbers to a decision.

This makes testing trivial. Construct a `PoolSnapshot` with the numbers you want, call `evaluate`, assert the decision. No mocking, no setup, no Quarkus context. The `TargetTrackingPolicy` tests are eight assertions against different fill ratios — each one a pure input-output check.

## Two things the design review caught

Claude's design review (three rounds, standard depth) caught a real correctness bug: `StepScalingPolicy` iterated steps in descending threshold order and returned on the first scale-in match. That meant graduated scale-in steps — mild response at 30% utilization, aggressive response at 10% — would always fire the mildest step first. The aggressive step was dead code.

The fix: track the last matching scale-in step across the full iteration, then return it after the loop. Scale-out still returns immediately on first match (descending order means the most urgent threshold wins). Scale-in keeps going to find the most aggressive match.

The second catch was subtler: `ConcurrentHashMap.computeIfAbsent` doesn't store null values. When a custom CDI policy bean wasn't found, `resolveCustomPolicy` returned null, `computeIfAbsent` stored nothing, and the next tick retried the CDI lookup — every 15 seconds, with a warning log each time. The fix was a `failedPolicyLookups` sentinel set that short-circuits after the first failure.

## Sealed config hierarchy

The YAML configuration maps to a sealed interface hierarchy — `TargetTrackingConfig`, `StepConfig`, `CustomScalingConfig`, `NoScalingConfig`. Each variant carries only its relevant fields. No `scaling:` section in YAML means `NoScalingConfig.INSTANCE` — fully backward compatible with existing pool definitions.

The sealed hierarchy pays off in `ScalingScheduler.createPolicy()` — exhaustive pattern matching means the compiler enforces that every config variant produces a policy. Add a new config type without handling it, and the code won't compile.

## What this opens up

The `PoolSnapshot` record has a `DemandMetrics` inner record — eviction count, exhaustion count, acquire count per tick interval. The built-in policies only use `fillRatio()` today, but the demand metrics are already flowing. A future policy could react to exhaustion spikes directly — "three rejections in the last tick, scale out immediately" — without changing the SPI.

The `CustomScalingConfig` variant resolves CDI `@Named` beans at runtime. Anyone can drop a `@Named("demand-pressure") @ApplicationScoped` class implementing `ScalingPolicy` into the classpath, reference it in YAML as `type: demand-pressure`, and the scheduler picks it up. No framework changes needed.
