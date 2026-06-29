---
layout: post
title: "The data was already there"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [provisioning, mesh, spi, platform-coherence]
series: issue-163-mesh-prompt-ops-config
---

The mesh prompt gap turned out to be embarrassingly simple. `MeshSystemPromptTemplate` generates a per-case prompt — channels, prior workers, participation mode — and `ClaudonyReactiveWorkerContextProvider` stores it in `WorkerContext.properties("systemPrompt")`. The engine passes that `WorkerContext` to the provisioner inside `ProvisionContext`. The provisioner had the prompt in its hands the whole time. It just never looked at it.

I'd expected to find a missing data path — something needed to be threaded from the context provider to the provisioner. Instead, `ProvisionContext.workerContext()` was right there in the method signature, carrying exactly the right data, ignored since the provisioner was written. The fix was six lines of Optional chaining.

What was less obvious was the prompt layering. A worker can have up to three prompt sources: Claude Code's default (Identity), an operator's static config (Persona), and the mesh prompt (Context). These are orthogonal layers — the operator might replace the entire default system prompt for a specialised agent while the mesh prompt provides case-specific operational context. Both should reach the CLI independently.

The existing `WorkerCommandBuilder` treated `--system-prompt` and `--append-system-prompt` as mutually exclusive — if/else, never both. That was wrong regardless of the mesh work. The CLI composes them: `--system-prompt` sets the base, `--append-system-prompt` appends to it. Removing the mutual exclusion and adding a `dynamicAppendPrompt` parameter for per-provisioning context cleaned up the layering.

Enabling delivery also surfaced a latent bug. The engine passes `workerId=null` to `buildContext()` because the session ID isn't generated until provisioning — later in the pipeline. Java's `"register(\"" + null + "\""` silently produces `register("null", ...)`. This had been harmless because the prompt was never delivered. The moment it became live, the template was producing instructions with a literal string "null" as the worker's registration ID. A null guard with a `<your-session-id>` placeholder fixed it, but the pattern is worth noting: enabling a previously-dead output path exposes every silent null coercion in the template chain.

The second item — `ProvisionerConfigRegistry` — started as "read worker config from casehub-ops" and turned into a platform SPI question. Claudony has `ProviderConfigSource`, OpenClaw has `AgentProviderConfigSource`. Both do the same thing: per-agent config lookup for their respective provisioners. That's the N×M problem the Platform Coherence Protocol flags. The solution was a shared `ProvisionerConfigRegistry` in `casehub-engine-api` — `configFor(providerName, agentId) → Map<String, Object>` — with each provisioner keeping its own typed adapter. Claudony's `CompositeProviderConfigSource` checks the registry first, falls back to `application.properties`.

The design review caught something I'd missed: I put the NoOp `@DefaultBean` in `casehub-engine-api`. That's a Tier 1 pure-Java module — no `quarkus-arc` on the classpath. Every existing NoOp in the engine follows the split: interface in `api/`, `@DefaultBean` in `runtime/`. The fix was obvious once pointed out, but I wouldn't have caught it without the adversarial pass.

The real win from the ops item isn't the registry itself — it's that the architecture is ready. When casehub-ops ships a `DeploymentProvisionerConfigRegistry @Alternative @Priority(1)`, every provisioner in the platform gets deployment-topology-driven config automatically. The N×M problem is gone.
