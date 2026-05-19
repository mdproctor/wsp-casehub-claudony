# Handover — 2026-05-19

**Head commit (project):** `bb72fc0` — docs(claude): sync test conventions after #115  
**Branch:** `main` — #115 closed, epic-io-thread-safety merged

---

## What happened this session

**Issue closed:** #115 — `ClaudonyWorkerContextProvider.buildContext()` called on Vert.x IO thread  
**Epic closed:** `epic-io-thread-safety` — merged to main via Option 1 (local merge)

**Core work — reactive SPI migration:**
- `CaseLineageQuery` → `Uni<List<WorkerSummary>>`; `JpaCaseLineageQuery` uses CDI self-injection + virtual-thread offload for `@Transactional`
- New `ClaudonyReactiveCaseChannelProvider` implements `ReactiveCaseChannelProvider` — injects `ReactiveChannelService` + `ReactiveMessageService` directly (not via `QhorusMcpTools`)
- New `ClaudonyReactiveWorkerContextProvider` implements `ReactiveWorkerContextProvider` — parallel `Uni.combine().all()` for lineage + channels
- New `ClaudonyReactiveWorkerProvisioner` implements `ReactiveWorkerProvisioner` — tmux offloaded to worker pool
- `quarkus.datasource.qhorus.reactive=true` + `quarkus-reactive-h2-client` activated in app
- `CaseContextChangedEventHandler.tryProvision()` in casehub-engine now returns `Uni<Void>`, injects reactive SPIs — committed and installed locally

**Acceptance criterion met:** `CaseEngineRoundTripTest` removes `@InjectMock` — real `ClaudonyReactiveWorkerContextProvider` exercised. Test passes.

**Architecture decisions during brainstorming:**
- No parallel blocking stacks in Claudony (unlike qhorus ADR-0003 which is for a foundation layer)
- `H2's limitations do not drive architecture` — one reactive implementation, `quarkus-reactive-h2-client` for tests
- `ClaudonyChannelBackend` (inbound display) and `ClaudonyReactiveCaseChannelProvider` (outbound management) are orthogonal concerns
- Multi-node fleet channel delivery is a separate problem (claudony#118)

**Platform docs updated:** `PLATFORM.md` (ChannelBackend/MessageObserver rows + 3 boundary rules), qhorus `messaging-architecture.md` (topology guidance: LOCAL vs CLUSTER vs fleet)

**Issues filed this session:**
- claudony#116 — verify reactive PostgreSQL path
- claudony#117 — `ClaudonyChannelBackend` for conversation panel
- claudony#118 — multi-node fleet channel delivery
- claudony#119 — `MeshResource` Uni<T> refactoring (replace `@Blocking` with proper return types)
- claudony#120 — `openChannel` race condition in `ClaudonyReactiveCaseChannelProvider`
- qhorus#141 — updated: add `quarkus-reactive-h2-client` investigation
- qhorus#161 — `ReactiveChannelService.findByNamePrefix()` for efficient per-case listing
- qhorus#168 — reduce `selected-alternatives` maintenance burden

**Garden:** 3 new entries (CDI self-injection @Transactional, RESTEasy Reactive .await() without @Blocking, virtual-thread offload technique) + REVISE to GE-20260417-c59817  
**Protocol:** PP-20260519-5f6d9f — `claudony-reactive-spi-variants.md`  
**Blog:** `2026-05-19-mdp01-what-the-mock-was-hiding.md`

---

## Test count

**130 in claudony-casehub, 4 in claudony-core — confirmed passing.** Full app module count pending `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` from user terminal (requires tmux on PATH). Previous baseline: 479 (2026-05-15).

---

## Open issues

- **#119** — `MeshResource` Uni<T> refactoring (currently uses `@Blocking` as stopgap)
- **#120** — `openChannel` concurrent init race condition
- **#116** — reactive PostgreSQL path verification
- **#117** — `ClaudonyChannelBackend` (HumanParticipatingChannelBackend) — **fully unblocked** (qhorus#131 + qhorus#153 both closed)
- **#118** — multi-node fleet channel delivery
- **#113** — `CaseHub.startCase()` IO-thread blocker (blocked on engine)
- **#105** — MCP endpoint separation
- **#99 epic** — channel gateway integration maturity (qhorus#131 now closed — reassess blockers)

*Full list: `gh issue list --repo casehubio/claudony --state open`*

---

## Immediate next

1. **Run full test suite from terminal** — `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` — get app count, update CLAUDE.md
2. **Pick up next issue** — #119 (Uni<T> refactoring for MeshResource, self-contained) or wait for upstream unblocking on #113/#99

**Before starting:** invoke `work-start` — if no active epic, invoke `/epic begin` first.

---

## Key references

- Plan: `plans/2026-05-18-io-thread-safety.md`
- Spec: `specs/epic-io-thread-safety/2026-05-18-io-thread-safety-design.md`
- Blog: `blog/2026-05-19-mdp01-what-the-mock-was-hiding.md`
- Protocol: `docs/protocols/casehub/claudony-reactive-spi-variants.md` (parent repo)
- Garden: GE-20260518-069f64, GE-20260518-e4fa52, GE-20260518-bee1b3
