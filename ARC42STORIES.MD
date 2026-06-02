# claudony — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-qhorus, casehub-engine SPIs, casehub-work. Integration layer — depends on most of the foundation stack.
**Consumed by:** Application-tier harness apps (via WorkerProvisioner SPI — provides Claude CLI sessions as CaseHub workers)
**Depends on:** casehub-qhorus (ChannelService, MessageService), casehub-engine SPIs (WorkloadProvider, WorkerProvisioner), casehub-work (WorkItemService).

---

## §1 Introduction and Goals

### Description

`claudony` wires Claude CLI sessions into the CasHub accountability framework. It implements foundation SPIs to provision remote Claude sessions as CaseHub workers — dispatching COMMAND obligations to real Claude agents and bridging their DONE/DECLINE responses back into the case context.

Core capabilities: `ClaudonyWorkerProvisioner` (implements casehub-engine's WorkerProvisioner SPI — launches Claude CLI sessions on demand), `PersonaMapping` (maps Claude session identity to CaseHub ActorId — bridges `InstanceActorIdProvider` SPI), `ClaudonyConfig` (session throttle, model selection, workspace configuration), unified ecosystem dashboard (GitHub PRs, issues, CI status, trust scores, active cases across all casehubio repos), claudony agent mesh framework (normative 3-channel layout: work/observe/oversight; 4-layer accountability framework).

Claudony is the integration layer — it knows about both the Claude Code runtime (sessions, models, hooks) and the CaseHub SPI surface. Application-tier apps get real LLM agents by including claudony as a dependency; without it, agents are stub implementations.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| SPI implementation | ClaudonyWorkerProvisioner satisfies the engine WorkerProvisioner SPI — application-tier apps get real agents without code changes |
| Identity bridge | PersonaMapping ensures every Claude session has a stable, auditable ActorId in the ledger — no anonymous agent actions |
| Dashboard coherence | The unified dashboard reads from all casehubio repos — provides a single observability surface for the full ecosystem |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:claudony:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** claudony owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `WorkerProvisioner` | casehub-engine SPI implemented by claudony — provisions Claude CLI sessions as CaseHub workers on COMMAND dispatch. |
| `PersonaMapping` | Maps Claude session identity (model, persona name) to a stable casehub ActorId for ledger audit. |
| `Agent mesh framework` | Normative 3-channel layout (work/observe/oversight) + 4-layer accountability framework for Claude agent coordination. See docs/repos/claudony.md. |
| `ClaudonyConfig` | Session throttle (SpawnThrottle), model selection, workspace path, boot sequence configuration. |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
