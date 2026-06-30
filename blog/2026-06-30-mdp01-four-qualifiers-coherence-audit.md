---
layout: post
title: "Four qualifiers and a coherence audit"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [Claudony]
tags: [cdi, platform-coherence, spi]
---

## The qualifier that ate the test suite

Engine shipped a cross-repo commit overnight — migrating `ClaudonyWorkerExecutionManager` to the new `@WorkerBackend` CDI qualifier. The commit fixed all four production injection sites in Claudony. It missed all four test injection sites.

The failure mode is specific and worth understanding. CDI §2.3.1 says: when you add any qualifier to a bean, the implicit `@Default` qualifier disappears. Every `@Inject` and `@InjectMock` site that doesn't carry the new qualifier now resolves to nothing. The error surfaces at Quarkus augmentation time — compilation succeeds, then augmentation explodes with `UnsatisfiedResolutionException`. In a single repo, your IDE catches this immediately. In a multi-repo platform, the upstream committer fixes what they can see. Downstream repos break silently until someone runs their tests.

The fix was four lines — add `@WorkerBackend` to the `@Inject` and `@InjectMock` declarations in `WorkerExitRecoveryIntegrationTest`, `CaseEngineRoundTripTest`, `ClaudonyLedgerEventCaptureSignalTest`, and `ClaudonyLedgerEventCaptureTest`. The pattern is mechanical; the blind spot is structural.

## Tracing ProvisionerConfigRegistry across the platform

With time freed up while ops and openclaw sessions worked on their own branches, I ran a coherence audit on the `ProvisionerConfigRegistry` SPI — the shared config lookup that engine#584 introduced and Claudony#164 consumed.

The question: do all three consumer repos (Claudony, OpenClaw, casehub-ops) use the SPI consistently, or have parallel implementations crept in?

The answer turned out to be cleaner than expected. All three repos follow the same structural pattern — registry-first lookup with local-config fallback, returning a provider-specific typed config:

- **Claudony** → `CompositeProviderConfigSource` calls `registry.configFor("claudony", agentId)`, falls back to `CaseHubConfig.Workers`
- **OpenClaw** → `OpenClawAgentConfigResolver` calls `registry.configFor("openclaw", agentId)`, falls back to `OpenClawCasehubConfig`
- **Ops** → implements the registry itself as `@Alternative @Priority(1)`, wrapping `DeploymentProviderConfigStore`

Each consumer has a different output type — `ClaudonyProviderConfig`, `AgentConfig`, `ProviderConfig` — because each provider needs different fields. The adapters are private implementation details, not shared abstractions. The SPI is the shared contract and all three repos respect it.

Two issues turned out to be already done: Claudony #165 (migrate to ProvisionerConfigRegistry) was delivered by #164 and never closed — closed it. OpenClaw #56 (migrate AgentProviderConfigSource) had been completed with commits visible in git history — the old class was already deleted.

The kind of session where the most useful thing you do is confirm that nothing needs doing.
